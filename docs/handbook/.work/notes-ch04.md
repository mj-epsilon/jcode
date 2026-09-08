# Phase 2 Notes — Chapter 4: Structured Concurrency & Cancellation

Verification status: every citation below was re-derived by personally
re-opening `crates/jcode-app-core/src/server/runtime.rs` (full file, 484
lines) and `crates/jcode-agent-runtime/src/lib.rs` (full file, 283 lines) on
2026-08-12, independent of the Phase 1 map's paraphrase. **Every line range
the map gave for these two files checked out exactly** — no corrections
needed (contrast with Chapters 2/3, where the map itself found and flagged
off-by-a-few-lines errors). Where I quote a doc comment, I re-read it
verbatim from the file, not from the map.

---

## Part A: `RuntimeTaskScope` — structured concurrency for the server

### The problem, in plain English (say this before any Rust)

A server that handles many client connections wants to spawn a background
task per connection (and per accept-loop). The classic bug in this shape:
you spawn a task, drop the handle because you don't need its result, and
now you have no way to (a) know if it's still running, or (b) tell it to
stop when the server shuts down. Tasks like this "leak" — they keep running
against a server that's supposedly gone, potentially against connections,
files, or state that no longer make sense.

"Structured concurrency" is the discipline of never letting a spawned task
outlive the scope that spawned it — the parent must be able to enumerate
every child it started and wait for (or cancel) every one of them before
the parent itself is considered done. `RuntimeTaskScope` is jcode's
hand-rolled implementation of that discipline for the server's connection
tasks, built from two off-the-shelf tokio primitives combined:

- a **join set** — a live collection of "tasks I've spawned, that I can
  wait on all at once, or one at a time as they finish" (this is
  `tokio::task::JoinSet`)
- a **cancellation token** — a shareable "please stop" flag with parent/child
  propagation, so one shutdown signal fans out to every spawned task (this is
  `tokio_util::sync::CancellationToken`)

### The struct and its design rationale

`crates/jcode-app-core/src/server/runtime.rs:33-37`
```rust
#[derive(Default)]
struct RuntimeTaskScope {
    cancellation: CancellationToken,
    tasks: Mutex<JoinSet<()>>,
}
```
Doc comment, `runtime.rs:27-32` (quote verbatim, strong "why" material):
> "Owns every connection task spawned by a server runtime. Dropping a
> `JoinHandle` detaches its task, so accepting a connection must not
> discard the handle. This scope gives the accept loops and their children
> one cancellation boundary and lets server shutdown wait until all
> children have observed cancellation and released their resources."

**[RUST → plain English]** `JoinSet<()>` is a collection you can add
running tasks to and later drain — either "give me whichever finishes
next" (`try_join_next()`/`join_next()`) or "wait for all of them." It's
like `Promise.all`, except tasks can be added to the set dynamically after
some are already running, and you can poll it incrementally instead of
only getting one combined result at the end. `CancellationToken` is a
cheap-to-clone "please stop" flag; calling `.child_token()` on it produces
a *linked* token that fires whenever its parent fires (but can also be
independently checked) — that's how one shutdown call fans out to every
spawned task without the scope having to iterate and notify each one by
hand.

**TS teaching equivalent**: `AbortController`/`AbortSignal`, but note one
real gap: `AbortSignal` has no built-in hierarchical `child_token()`.
Chaining is possible (listen for the parent's `abort` event and call
`.abort()` on a child controller) but it's manual, whereas
`CancellationToken::child_token()` is a first-class one-liner in Rust's
`tokio-util`. For the "join set" half, there's no single built-in — the
idiomatic TS shape is a `Set<Promise<void>>` you add to on spawn and
remove from on settle (via `.finally()`), or a small tracking array awaited
with `Promise.allSettled` at shutdown.

### `RuntimeTaskScope::spawn` — registering a task under the scope

`runtime.rs:40-59`
```rust
async fn spawn<F, Fut>(&self, task: F) -> bool
where
    F: FnOnce(CancellationToken) -> Fut,
    Fut: Future<Output = ()> + Send + 'static,
{
    if self.cancellation.is_cancelled() {
        return false;
    }
    let mut tasks = self.tasks.lock().await;
    while let Some(result) = tasks.try_join_next() {
        log_task_completion(result);
    }
    if self.cancellation.is_cancelled() {
        return false;
    }
    tasks.spawn(task(self.cancellation.child_token()));
    true
}
```
Plain English: before accepting a new task, it (1) refuses outright if the
scope is already shutting down, (2) opportunistically drains any
already-finished tasks out of the set (so the set doesn't grow unbounded —
"take out the trash while you're at the door" rather than a separate GC
pass), (3) double-checks cancellation again after that (in case shutdown
raced in), then (4) spawns, handing the new task its own linked child
token so it can watch for cancellation itself. Returns `bool` — whether the
task was actually accepted — so callers can react to "sorry, we're
shutting down" rather than silently losing work.

**[RUST → plain English]** The generic bound `F: FnOnce(CancellationToken)
-> Fut` reads as "give me a function that, when handed a cancellation
token, returns some future" — the scope calls this function itself so it
can inject the child token; callers never construct the token themselves.
`Fut: Future<Output = ()> + Send + 'static` are Rust's compile-time
guarantees that the resulting future is safe to run on any of tokio's
worker OS threads (`Send`) at any later time, not borrowing any local
stack data that could go away first (`'static`) — necessary because
tokio's scheduler is free to move/resume tasks on different threads. There
is no TS equivalent to this check; it doesn't exist as a *problem* in
single-threaded JS.

**TS teaching equivalent**: a `scope.spawn(fn)` method that (1) throws/
returns false if `controller.signal.aborted`, (2) prunes an internal
`Set` of settled promises, (3) calls `fn(controller.signal)` — passing the
same shared `AbortSignal` (no true "child" needed in JS since there's only
one thread to race against), (4) adds the resulting promise to the set.

### `RuntimeTaskScope::shutdown` — the deadlock-avoidance detail

`runtime.rs:61-74`
```rust
async fn shutdown(&self) {
    self.cancellation.cancel();
    // Drain the set before awaiting children. An accept task may already be
    // waiting to register a just-accepted connection; leaving the mutex
    // held while joining would deadlock that task. Once cancelled, any
    // late registration observes cancellation and is rejected.
    let mut tasks = {
        let mut owned_tasks = self.tasks.lock().await;
        std::mem::take(&mut *owned_tasks)
    };
    while let Some(result) = tasks.join_next().await {
        log_task_completion(result);
    }
}
```
Plain English, and *why this is a good teaching example*: this is a
concrete, named instance of a real concurrency bug class —
**lock-held-across-await deadlock**. If `shutdown` held the mutex on the
task set *while* also awaiting each child task to finish, and one of those
children was itself blocked trying to acquire that same mutex (to register
itself), the two would wait on each other forever. The fix here is
`std::mem::take` — swap the entire `JoinSet` out for an empty default one
while briefly holding the lock, then release the lock immediately, and
only *then* await the (now unlocked, locally-owned) set of tasks. This is
a "steal the collection out from under the lock, then work with your own
copy" pattern — one that's easy to explain to a non-Rust reader as "don't
hold the front door open while you wait for everyone still inside to leave
— take a copy of the guest list, close the door, then check people off
your copy."

**[RUST → plain English]** `std::mem::take(&mut *owned_tasks)` replaces
whatever's behind the mutable reference with `Default::default()` (an
empty `JoinSet` here) and hands back the *old* value — a cheap,
allocation-free swap, not a clone.

**TS teaching equivalent**: the direct analogue would be "empty the
tracking `Set` into a local array (`const mine = [...set]; set.clear();`)
before `await Promise.allSettled(mine)`" — but it's worth telling the
reader plainly that in single-threaded JS there is no possible deadlock
here in the first place, because there's no second thread that could be
concurrently blocked trying to acquire the same lock; a `Set` isn't even
lock-guarded. This is a good "why does Rust need this at all" moment: the
hazard is real only because tokio genuinely runs tasks on multiple OS
threads simultaneously.

### `RuntimeTaskScope::task_count` — `runtime.rs:76-79`, test-only helper. Not chapter-central, skip or one-line mention.

### `log_task_completion` — `runtime.rs:82-88`

Plain English: logs a task's error unless the error was just "it got
cancelled" (`error.is_cancelled()`) — i.e. cancellation itself is not
treated as a failure worth logging, only genuine panics/errors are.

### `ServerRuntime` composition — `runtime.rs:90-120`

The struct that holds the whole server's shared state, including `tasks:
Arc<RuntimeTaskScope>` at line 119. This is the bridge to Chapter 5 — most
of its other fields are `Arc<RwLock<...>>`/`Arc<Mutex<...>>` registries
(see Chapter 5 notes). For Chapter 4 the useful thing to show is just that
one scope instance is shared (via `Arc`) across every clone of
`ServerRuntime`, so every connection handler that gets a `ServerRuntime`
clone is registering its background work with the *same* scope and will be
cancelled/joined together at shutdown.

### Two different spawn styles, worth contrasting side by side

1. **Outside the scope, `tokio::spawn` directly** — `spawn_main_accept_loop`
   (`runtime.rs:156-185`) and `spawn_debug_accept_loop` (`187-219`). Each
   accept loop gets its own `child_token()` (line 158/193) but is spawned
   with a bare `tokio::spawn`, *not* `self.tasks.spawn(...)`. One sentence
   on why this is a deliberate, not sloppy, choice: an accept loop is a
   long-lived structural task tied 1:1 to the runtime's own lifetime (it's
   spawned once, from `from_server`-adjacent setup code, and its
   `JoinHandle` is returned to and held by the caller) rather than a
   dynamically-arriving unit of work the scope needs to track and
   opportunistically reap — so it's cancelled via the same token tree but
   tracked by its own returned handle instead of the scope's `JoinSet`.
   The `tokio::select!` inside the loop body (`runtime.rs:164-167`):
   ```rust
   let accepted = tokio::select! {
       _ = cancellation.cancelled() => break,
       accepted = listener.accept() => accepted,
   };
   ```
   races "a new connection arrived" against "we've been told to stop,"
   taking whichever happens first.

2. **Inside the scope, `self.tasks.spawn(...)`** — every *connection*
   handler goes through the scope: `spawn_client_task` (`266-280`),
   `spawn_gateway_client_task` (`282-296`), `spawn_debug_client_task`
   (`298-307`), and the generic `spawn_background_task` (`252-264`):
   ```rust
   pub(super) async fn spawn_background_task<Fut>(&self, task: Fut) -> bool
   where
       Fut: Future<Output = ()> + Send + 'static,
   {
       self.tasks
           .spawn(move |cancellation| async move {
               tokio::select! {
                   _ = cancellation.cancelled() => {}
                   _ = task => {}
               }
           })
           .await
   }
   ```
   This is the general-purpose "run this and cancel it on shutdown" entry
   point other server code calls when it wants a scoped background job
   without writing its own `select!`. Good single central example for the
   chapter: it shows the scope-spawn API used generically.

   `spawn_gateway_accept_loop` (`221-250`) is a third, hybrid case worth
   one line: it's an accept *loop* (like #1 above structurally) but *is*
   registered through `self.tasks.spawn` (like #2) — so the two styles
   aren't a strict "loops vs. connections" split, just two available tools
   used where each made sense; don't over-claim a clean rule the code
   itself doesn't enforce.

3. **Cooperative cancellation in the per-connection body** —
   `run_client_stream` (`333-391`) and `run_debug_stream` (`393-438`) each
   race the real work against the token:
   ```rust
   tokio::pin!(client);
   tokio::select! {
       result = &mut client => Some(result),
       _ = cancellation.cancelled() => None,
   }
   ```
   (`runtime.rs:376-379`, `432-435`). **[RUST → plain English, load-bearing
   for the chapter]**: Rust/tokio cancellation is *cooperative*, not
   *preemptive* — a task is not forcibly killed; it keeps running until it
   reaches an `.await` point that happens to be racing the cancellation
   token. This is exactly the `AbortSignal` model in JS/TS: code must
   itself check `signal.aborted` or pass the signal into an
   abort-aware API (like `fetch(url, { signal })`); nothing yanks control
   away from currently-executing synchronous code. Good one-sentence
   summary for the chapter: "cancellation in both worlds is a *request*
   the task has to notice, not a kill switch."

### Tests as design documentation — `runtime.rs:441-484`

`runtime_task_scope_cancels_and_joins_owned_tasks` spawns a task holding a
`DropFlag` (a wrapper whose `Drop` impl flips an `AtomicBool`), calls
`scope.shutdown()`, and asserts (a) the flag was dropped [proves the task
was actually torn down, not just marked cancelled and abandoned], (b) the
task count is back to 0, and (c) a `spawn` attempted *after* shutdown is
rejected (`assert!(!scope.spawn(...).await)`). Good "here's the contract,
proven" callout — this is exactly the shape of test a TS port would want
too (spawn one task that runs forever unless aborted, cancel the scope,
assert cleanup ran and post-shutdown spawns are refused).

---

## Part B: `InterruptSignal` — the agent's own cancel primitive

### The problem, in plain English

Separately from server-connection shutdown (Part A), jcode needs a way for
a user hitting Esc/Ctrl+C to cancel an in-flight agent turn — stop
streaming, stop a running tool — as fast as possible, without the agent
loop having to constantly poll "was I cancelled?" in a tight spin loop
(wasteful) nor miss a cancel that arrives at an inconvenient instant (a
correctness bug users would perceive as "my Esc didn't do anything").
`InterruptSignal` (`crates/jcode-agent-runtime/src/lib.rs`) is a
hand-rolled primitive built to satisfy both: cheap to check synchronously
from a hot loop, and awaitable asynchronously without busy-waiting.

Doc comment, `lib.rs:30-31` (quote verbatim):
> "Async-aware interrupt signal that combines AtomicBool (sync read) with
> tokio::Notify (async wake). Eliminates spin-loops during tool execution."

### The struct

`lib.rs:32-40`
```rust
#[derive(Clone)]
pub struct InterruptSignal {
    flag: Arc<std::sync::atomic::AtomicBool>,
    epoch: Arc<std::sync::atomic::AtomicU64>,
    notify: Arc<tokio::sync::Notify>,
}
```
**[RUST → plain English]** An `AtomicBool` is a boolean that many threads
can safely read/write at once without a lock — cheap, hardware-backed.
`tokio::sync::Notify` is async code's equivalent of a condition variable:
tasks can `.await` on `notified()` and be woken up when someone elsewhere
calls `notify_waiters()`. Combining the two gives two access styles for
the price of one signal: a synchronous, lock-free `is_set()` check for hot
paths, and an asynchronous `.await`-able `notified()` for code that wants
to sleep until cancelled rather than poll.

### Why there's a third field (`epoch`) — the interesting design decision

Field doc comment, `lib.rs:35-37`:
> "Monotonic fire counter. Lets owners of a timed/deferred reset detect
> that a *newer* fire landed in the meantime and skip the reset instead of
> erasing a cancel the target has not observed yet (issue #428)."

Plain English motivating example (this is worth including verbatim-ish in
the chapter, it's concrete): imagine the server schedules "reset this
signal in 500ms" after firing it once, to auto-clear a cancel. If the user
presses Esc *again* before that 500ms elapses, a naive `reset()` would
wipe out the second, newer cancel along with clearing the first one —
the agent would silently un-cancel itself out from under a user who just
cancelled it a second time. `reset_if_epoch` solves this by having the
resetter record which "version" (epoch number) of the signal it intends
to reset, and only actually clearing the flag if no newer `fire()` has
happened since.

`lib.rs:78-90`:
```rust
pub fn reset_if_epoch(&self, epoch: u64) -> bool {
    if self.epoch.load(std::sync::atomic::Ordering::SeqCst) != epoch {
        return false;
    }
    self.flag.store(false, std::sync::atomic::Ordering::SeqCst);
    if self.epoch.load(std::sync::atomic::Ordering::SeqCst) != epoch {
        // A newer fire raced with the reset; restore it.
        self.flag.store(true, std::sync::atomic::Ordering::SeqCst);
        self.notify.notify_waiters();
        return false;
    }
    true
}
```
Doc comment, `lib.rs:72-77`:
> "Reset the signal only if no newer `fire` happened since `epoch` was
> captured... If a racing fire lands between the epoch check and the
> reset, the fire is restored (flag re-set and waiters re-notified) so no
> cancel is ever silently erased."

Note the double-check pattern: it checks the epoch, clears the flag, then
checks the epoch *again* — because a fire could race in during the moment
between the flag write and the second check, and if it did, the function
puts the flag (and notification) back rather than leaving a lost cancel.

`fire()` itself, `lib.rs:51-55`:
```rust
pub fn fire(&self) {
    self.epoch.fetch_add(1, std::sync::atomic::Ordering::SeqCst);
    self.flag.store(true, std::sync::atomic::Ordering::SeqCst);
    self.notify.notify_waiters();
}
```
Always does all three steps (bump epoch, set flag, notify), so it doesn't
matter whether the eventual observer is a synchronous poller (`is_set()`)
or an async waiter (`notified()`) — both see a consistent fire.

### `notified()` — the second race this file guards against

`lib.rs:92-106`:
```rust
pub async fn notified(&self) {
    let mut notified = std::pin::pin!(self.notify.notified());
    notified.as_mut().enable();
    if self.is_set() {
        return;
    }
    notified.await;
}
```
Inline comment, `lib.rs:94-100` (paraphrased: quote closely, it's precise):
explicit `.enable()` registers this waiter with the underlying `Notify`
*before* checking the flag, specifically so a `fire()` landing in the
narrow window between "check the flag" and "start waiting" is not missed
— rather than relying on a version-specific tokio guarantee that
`notified()` futures are pre-registered from creation. Referenced
throughout as "issue #428" — a real, named bug this file's design and
its five tests exist to close out for good.

### Why this file is more elaborate than the "obvious" TS translation would be

This is genuinely one of the best "Rust vs. TS" teaching moments in the
whole codebase, worth calling out explicitly in the chapter prose (not
just in a gloss): the entire epoch/notify-registration dance exists to
close races that can only happen because Rust code can run **truly
concurrently on multiple OS threads**. In single-threaded JS, there is no
possible interleaving between "I checked `signal.aborted`" and "someone
else called `.abort()`" — the event loop guarantees one or the other
happens completely before the next JS statement runs. `AbortController`/
`AbortSignal` doesn't need an epoch counter or an explicit "enable before
check" step because the underlying execution model makes the race
physically impossible. A faithful teaching note here should say: *this is
one of the few places in jcode where the honest TS equivalent is
meaningfully simpler than the Rust, and the reason why is a great
one-paragraph explanation of what "single-threaded" actually buys you.*

### Tests as documentation, `lib.rs:142-283`

Five tests, each pinned to a specific guarantee (all read in full,
matches map's characterization exactly):
- `notified_future_receives_notify_waiters_from_creation` (153-173) —
  documents the tokio semantic the code relies on / hardens against
  changing.
- `fire_never_loses_wakeup_while_notified_races` (180-207) — a 2000-
  iteration hammer test on a real multi-threaded runtime
  (`worker_threads(2)`) specifically built to catch the lost-wakeup race
  probabilistically; good evidence this is a *real*, previously-observed
  bug class, not a hypothetical.
- `notified_returns_immediately_when_already_fired` (210-217).
- `reset_clears_fired_state` (220-229).
- `reset_if_epoch_skips_when_newer_fire_landed` (235-259) and
  `reset_if_epoch_never_erases_concurrent_fire` (263-282) — directly test
  the epoch-guarded reset scenario described above, the second one using a
  real spawned OS thread (`std::thread::spawn`) racing against
  `reset_if_epoch`, 2000 iterations.

### Other types in the same file (mention briefly, not chapter-central)

`SoftInterruptMessage`/`SoftInterruptSource` (`lib.rs:4-18`),
`SoftInterruptQueue` (21, `Arc<std::sync::Mutex<Vec<...>>>`),
`BackgroundToolSignal`/`GracefulShutdownSignal` (25, 28, both bare
`Arc<AtomicBool>` — simpler cousins of `InterruptSignal` without the
async-notify half, worth one sentence as "not every cancel signal in this
codebase needs the full epoch+Notify machinery — these two get by with
just the atomic flag because nothing needs to `.await` on them"). `
StreamError` (128-140) is unrelated (provider streaming error type) — do
not cite it in this chapter.

### How Part A and Part B relate (worth one paragraph in the chapter)

`RuntimeTaskScope`'s `CancellationToken` and `InterruptSignal` are **two
separate, differently-shaped cancellation primitives solving different
problems** in the same codebase, not one wrapping the other:
`CancellationToken` (from the `tokio-util` crate, off the shelf) shuts
down whole server connection tasks with built-in hierarchical
parent/child propagation; `InterruptSignal` (hand-rolled, in-house) is
specifically for the agent turn's own cancel/interrupt path and adds the
epoch-guarded reset semantics `CancellationToken` doesn't have (a
`CancellationToken` is one-shot — once cancelled it stays cancelled; it
has no "reset" concept at all, which is exactly why the agent needed its
own primitive rather than reusing `CancellationToken`). This contrast is
itself worth a sentence in the chapter: not every cancellation need in the
codebase is served by the same primitive, and the reasons why map cleanly
to explaining each primitive's shape.

**Open question (flag explicitly, don't guess):** I did not trace where
`InterruptSignal` is actually wired into the agent turn/tool-execution
loop (e.g. which function calls `.fire()` on Esc/Ctrl+C, or which loop
calls `.notified()`/`.is_set()`). The doc comments make the *intent* clear
("Eliminates spin-loops during tool execution," issue #428 references
"agent stream loop, tool-wait select") but I have not opened the call
sites myself. If Phase 3 wants to narrate "here's where this actually gets
used," that needs a fresh grep/read pass — don't invent a call site.

---

## Snippet-selection recommendation for Phase 3 (which ones to actually feature)

Given the "short handbook" budget, I'd pick at most 4-5 code blocks total
for this chapter:
1. `RuntimeTaskScope` struct + doc comment (`runtime.rs:27-37`) — sets up
   the vocabulary.
2. `RuntimeTaskScope::shutdown` (`runtime.rs:61-74`) — the
   lock-across-await deadlock-avoidance story; best single "here's a real
   subtle bug and how it's avoided" moment in the file.
3. The `tokio::select!` cancellation race in `run_client_stream`
   (`runtime.rs:376-379`) — shortest, clearest illustration of cooperative
   cancellation, directly parallel to `AbortSignal`.
4. `InterruptSignal` struct + `fire()` (`lib.rs:32-40`, `51-55`) — sets up
   the AtomicBool+Notify combo.
5. `reset_if_epoch` (`lib.rs:78-90`) — the standout "why is this so
   careful" exhibit and the best Rust-vs-TS contrast moment in either
   chapter's material; I'd make this the emotional centerpiece of the
   chapter if only one deep-dive snippet is allowed.

## TS-equivalent primitive mapping (summary table for Phase 3)

| Rust piece | file:line | TS teaching equivalent | Note |
| --- | --- | --- | --- |
| `CancellationToken` + `.child_token()` | `runtime.rs:35, 57` | `AbortController`/`AbortSignal`, manually chained for "child" behavior | Rust's child-token hierarchy is built-in; JS needs manual event forwarding to get the same fan-out |
| `JoinSet<()>` | `runtime.rs:36` | `Set<Promise<void>>` you add to on spawn, prune via `.finally()`, drain with `Promise.allSettled` at shutdown | No single built-in equivalent |
| `tokio::spawn` (fire-and-forget, outside scope) | `runtime.rs:159, 194` | calling an async function without `await` (a "floating promise") | Same tradeoff: errors/completion invisible to the caller unless separately reported |
| `tokio::select!` racing work vs. cancellation | `runtime.rs:164-167, 376-379` | `Promise.race([...])`, or passing `AbortSignal` into an abort-aware API like `fetch` | Cooperative in both models — nothing preempts running sync code |
| `std::mem::take` to release a lock before awaiting | `runtime.rs:67-70` | copy items out of a `Set`/array and clear it before `await`ing them | The *hazard* being avoided (lock-held-across-await deadlock) doesn't exist in single-threaded JS at all — worth saying explicitly |
| `AtomicBool` (sync, lock-free flag) | `lib.rs:34` | a plain boolean variable | No atomicity concept needed in JS — nothing runs concurrently within one turn of the event loop |
| `tokio::sync::Notify` | `lib.rs:39` | conceptually an `EventEmitter`/condition-variable; in practice `AbortSignal`'s `abort` event covers the same "wake all awaiters" need | — |
| `InterruptSignal` (AtomicBool + epoch + Notify) as a whole | `lib.rs:32-40` | `AbortController` + a manual "generation" counter if you need epoch-guarded resets (rare in JS; usually unnecessary) | The epoch/lost-wakeup machinery is solving a true-multithreading problem that mostly does not exist in JS — flag this explicitly as a place Rust is *more* complex than idiomatic TS, and explain why |

## Verification summary

- All `runtime.rs` citations (33-37, 40-59, 61-74, 76-79, 82-88, 90-120,
  156-185, 187-219, 221-250, 252-264, 266-307, 309-311, 333-391, 393-438,
  441-484) confirmed by direct read of the full 484-line file. No
  discrepancies from the map found.
- All `crates/jcode-agent-runtime/src/lib.rs` citations (32-40, 43-49,
  51-55, 57-59, 61-63, 68-70, 78-90, 92-106, 108-110, 115-117, 142-283)
  confirmed by direct read of the full 283-line file. No discrepancies
  from the map found.
- Open question flagged above: exact call sites that invoke
  `InterruptSignal::fire()`/`notified()`/`is_set()` from the agent
  turn/tool loop were not traced in this pass.
