# Chapter 4 — Structured Concurrency & Cancellation

Every agent swarm eventually asks the same two questions: *when I spawn
background work, how do I know it actually stops when I tell it to?* and
*when a user hits cancel, how fast and how reliably does that reach the
work in flight?*

jcode answers these with two separate, hand-built primitives that live at
different layers of the system:

- `RuntimeTaskScope` — makes sure the server never loses track of a
  background task it spawned, and can shut all of them down cleanly.
- `InterruptSignal` — lets a user's Esc/Ctrl+C reach a running agent turn
  fast, without the agent loop wasting cycles constantly polling "am I
  cancelled yet?"

Neither is exotic. Both are built from a small number of off-the-shelf
concurrency primitives, combined in a way that closes specific, nameable
bugs. That combination — and *why* it's needed at all — is what this
chapter teaches.

## Part A: `RuntimeTaskScope` — never losing track of a spawned task

### The problem

Picture a server that accepts many client connections and spawns a
background task to handle each one. The obvious way to spawn a task is
also the easiest way to create a bug: you fire off the task, you don't
need its result right this second, so you throw away the handle you'd use
to check on it. Now nobody owns that task. Nobody can ask "are you still
running?" and nobody can tell it to stop when the server shuts down. The
task keeps going — against a server that, as far as anyone else is
concerned, is already gone.

**Structured concurrency** is the fix as a discipline: a parent that
spawns child tasks must be able to enumerate every child it has spawned,
and must wait for (or forcibly stop) every one of them before the parent
itself is considered finished. `RuntimeTaskScope` is jcode's implementation
of that discipline for the server's connection-handling tasks.

It's built from two ingredients:

- a **join set** — a live collection of "tasks I've spawned that I can
  wait on, together or one at a time" (Rust's `tokio::task::JoinSet`)
- a **cancellation token** — a cheap, shareable "please stop" flag that
  can propagate from a parent to every child it hands a copy to (Rust's
  `tokio_util::sync::CancellationToken`)

Here's the struct, with its own doc comment left in because it states the
danger plainly:

```rust
// jcode: crates/jcode-app-core/src/server/runtime.rs:27-37
/// Owns every connection task spawned by a server runtime.
///
/// Dropping a `JoinHandle` detaches its task, so accepting a connection must not
/// discard the handle. This scope gives the accept loops and their children one
/// cancellation boundary and lets server shutdown wait until all children have
/// observed cancellation and released their resources.
#[derive(Default)]
struct RuntimeTaskScope {
    cancellation: CancellationToken,
    tasks: Mutex<JoinSet<()>>,
}
```

A `JoinSet<()>` behaves a bit like `Promise.all`, except you can keep
adding tasks to it after others are already running, and you can drain it
incrementally ("give me whichever one finished next") instead of only
getting one combined result at the very end. `CancellationToken` is cheap
to clone, and calling `.child_token()` on one produces a *linked* token
that fires automatically whenever its parent fires — that's how one
shutdown call reaches every spawned task without the scope having to loop
over them and notify each one by hand.

**TS teaching equivalent:** `AbortController`/`AbortSignal` cover the
"please stop" half, but there's a real gap worth naming: `AbortSignal` has
no built-in `child_token()`. Chaining parent-to-child abort behavior in JS
means manually listening for the parent's `abort` event and calling
`.abort()` on a child controller yourself. For the "join set" half, there's
no single built-in — the idiomatic shape is a `Set` of in-flight promises
you add to on spawn and remove from as they settle.

### Registering a task under the scope

```rust
// jcode: crates/jcode-app-core/src/server/runtime.rs:40-59
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

Before accepting a new task it: refuses outright if the scope is already
shutting down; opportunistically drains any already-finished tasks out of
the set (housekeeping done on the way in, rather than a separate cleanup
pass); checks cancellation *again*, in case shutdown started while it was
draining; then spawns, handing the new task its own linked child token so
it can watch for cancellation itself. It returns a `bool` — whether the
task was actually accepted — so a caller can react to "sorry, we're
shutting down" instead of silently losing the work.

(One Rust-specific detail, for context and then set aside: the bound
`Fut: Future<Output = ()> + Send + 'static` is the compiler's guarantee
that the task is safe to run on any of the runtime's worker OS threads and
doesn't secretly depend on local stack data that might disappear first.
This isn't a *problem* that exists in single-threaded JS at all — there's
nothing to guarantee.)

```typescript
// idiomatic TS equivalent — this code does not exist in jcode
class RuntimeTaskScope {
  private controller = new AbortController();
  private tasks = new Set<Promise<void>>();

  async spawn(task: (signal: AbortSignal) => Promise<void>): Promise<boolean> {
    if (this.controller.signal.aborted) return false;

    const promise = task(this.controller.signal).finally(() => {
      this.tasks.delete(promise);
    });
    this.tasks.add(promise);
    return true;
  }

  async shutdown(): Promise<void> {
    this.controller.abort();
    const inFlight = [...this.tasks];
    await Promise.allSettled(inFlight);
  }
}
```

### The deadlock this code was written to avoid

This is the best single "here's a real, subtle bug and the fix for it"
moment in the file:

```rust
// jcode: crates/jcode-app-core/src/server/runtime.rs:61-74
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

If `shutdown` held the lock on the task set *while also* waiting for each
child task to finish, and one of those children was itself blocked trying
to acquire that same lock (to register itself as a new task), the two
would wait on each other forever — a lock-held-across-await deadlock. The
fix, `std::mem::take(&mut *owned_tasks)`, swaps the whole `JoinSet` out
for an empty one while briefly holding the lock, then releases the lock
immediately and works with its own private copy from then on. It's the
concurrency equivalent of "don't hold the front door open while everyone
still inside leaves — take a copy of the guest list, close the door, then
check people off your copy."

**Worth saying plainly:** this exact hazard doesn't exist in single-threaded
JS. There's no second thread that could simultaneously be blocked trying to
acquire the same lock, and a plain `Set` isn't lock-guarded at all. The
Rust code needs this maneuver only because tokio genuinely runs tasks on
multiple OS threads at once.

```typescript
// idiomatic TS equivalent — this code does not exist in jcode
async function shutdown(scope: RuntimeTaskScope) {
  // No deadlock risk to design around here: JS has no second thread that
  // could be mid-way through acquiring a lock this code holds.
  const inFlight = [...scope["tasks"]];
  scope["tasks"].clear();
  await Promise.allSettled(inFlight);
}
```

### Cancellation is cooperative, not preemptive

Both the accept loops and the per-connection handlers race the actual
work against the cancellation token with `tokio::select!`:

```rust
// jcode: crates/jcode-app-core/src/server/runtime.rs:376-379
tokio::pin!(client);
tokio::select! {
    result = &mut client => Some(result),
    _ = cancellation.cancelled() => None,
}
```

This is the load-bearing idea for the whole chapter: **cancellation here is
a request, not a kill switch.** Nothing forcibly stops the running task —
it keeps going until it reaches a point where it's `.await`ing something,
racing that against "has cancellation fired." This is exactly the mental
model behind `AbortSignal` in JS: code has to check `signal.aborted`
itself, or hand the signal to something abort-aware like `fetch(url, {
signal })`. Nothing yanks control away from code that's mid-execution.

```typescript
// idiomatic TS equivalent — this code does not exist in jcode
async function runConnection(signal: AbortSignal, client: Promise<Result>) {
  return Promise.race([
    client.then((result) => ({ kind: "done" as const, result })),
    new Promise<{ kind: "cancelled" }>((resolve) => {
      signal.addEventListener("abort", () => resolve({ kind: "cancelled" }), {
        once: true,
      });
    }),
  ]);
}
```

Two spawn styles show up side by side in the file, worth a one-line note
so it doesn't look like a rule the code doesn't actually enforce: the
long-lived accept loops (`runtime.rs:156-185`, `187-219`) are spawned
directly with `tokio::spawn` and tracked by the caller's own returned
handle, while every per-connection handler goes through
`self.tasks.spawn(...)` so the scope can track and reap it. One accept
loop (`spawn_gateway_accept_loop`, `runtime.rs:221-250`) is registered
through the scope too — so it's not a strict "loops vs. connections"
split, just two available tools used where each made sense.

## Part B: `InterruptSignal` — cancelling an agent turn fast

### The problem

Separate from server shutdown, jcode needs a way for a user hitting
Esc/Ctrl+C to cancel an in-flight agent turn — stop streaming, stop a
running tool — as fast as possible. Two failure modes to avoid: making the
agent loop constantly ask "was I cancelled?" in a tight spin loop (wastes
CPU), or missing a cancel that arrives at an inconvenient instant (a real
bug users would feel as "my Esc didn't do anything"). `InterruptSignal` is
built to satisfy both — cheap to check synchronously from a hot loop, and
awaitable asynchronously without busy-waiting.

Its own doc comment states the design directly:

> "Async-aware interrupt signal that combines AtomicBool (sync read) with
> tokio::Notify (async wake). Eliminates spin-loops during tool execution."
> — `crates/jcode-agent-runtime/src/lib.rs:30-31`

```rust
// jcode: crates/jcode-agent-runtime/src/lib.rs:32-40
#[derive(Clone)]
pub struct InterruptSignal {
    flag: Arc<std::sync::atomic::AtomicBool>,
    epoch: Arc<std::sync::atomic::AtomicU64>,
    notify: Arc<tokio::sync::Notify>,
}
```

An `AtomicBool` is a boolean that many threads can safely read and write
without a lock. `tokio::sync::Notify` is async code's version of a
condition variable — tasks can `.await` a call to `notified()` and get
woken up when someone calls `notify_waiters()`. Together they give two
ways to check the same signal: a synchronous, lock-free `is_set()` for hot
paths, and an async, sleep-until-woken `notified()` for code that would
rather not poll.

```rust
// jcode: crates/jcode-agent-runtime/src/lib.rs:51-55
pub fn fire(&self) {
    self.epoch.fetch_add(1, std::sync::atomic::Ordering::SeqCst);
    self.flag.store(true, std::sync::atomic::Ordering::SeqCst);
    self.notify.notify_waiters();
}
```

`fire()` always does all three steps together, so it doesn't matter
whether the eventual observer is polling synchronously or waiting
asynchronously — both see a consistent cancel.

### The interesting part: why there's a third field

The `epoch` field is the one piece of this file worth slowing down for.
Its own doc comment:

> "Monotonic fire counter. Lets owners of a timed/deferred reset detect
> that a *newer* fire landed in the meantime and skip the reset instead of
> erasing a cancel the target has not observed yet (issue #428)."
> — `crates/jcode-agent-runtime/src/lib.rs:35-37`

Concrete scenario: imagine something schedules "auto-clear this cancel in
500ms" right after firing it. If the user presses Esc *again* before those
500ms are up, a naive `reset()` would wipe out both the second, newer
cancel *and* the auto-clear's own bookkeeping — the agent would silently
un-cancel itself out from under a user who had just cancelled it a second
time. `reset_if_epoch` closes that gap: the resetter records which
"version" of the signal it means to clear, and only actually clears the
flag if no newer `fire()` happened since.

```rust
// jcode: crates/jcode-agent-runtime/src/lib.rs:78-90
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

Notice the double-check: it reads the epoch, clears the flag, then reads
the epoch *again* — because a new `fire()` could land in the gap between
those two steps, on another thread, at any moment. If that happens, the
function puts the flag (and the notification) back rather than letting a
cancel get silently lost.

There's a companion race guarded in `notified()`:

```rust
// jcode: crates/jcode-agent-runtime/src/lib.rs:92-106
pub async fn notified(&self) {
    let mut notified = std::pin::pin!(self.notify.notified());
    notified.as_mut().enable();
    if self.is_set() {
        return;
    }
    notified.await;
}
```

The explicit `.enable()` registers this waiter with `Notify` *before*
checking the flag, specifically so a `fire()` landing in the narrow window
between "check the flag" and "start waiting" isn't missed. The file's own
comments trace this back to a real, named bug ("issue #428"), and one of
its tests hammers the race 2,000 times on a real multi-threaded runtime to
catch it probabilistically — this is documentation of a bug that actually
happened, not a hypothetical.

**Why this is one of the best Rust-vs-TS contrasts in the whole codebase:**
the entire epoch/enable-before-check dance exists to close races that can
only happen because Rust code genuinely runs on multiple OS threads at
once. In single-threaded JS, there is no possible interleaving between "I
checked `signal.aborted`" and "someone else called `.abort()`" — the event
loop guarantees one completes before the other's code runs at all.
`AbortController` doesn't need an epoch counter or an "enable before
check" step because the execution model makes the race physically
impossible. This is a case where the honest TS equivalent is genuinely
*simpler* than the Rust — not a simplification for teaching purposes, but
a real reflection of what single-threadedness buys you.

```typescript
// idiomatic TS equivalent — this code does not exist in jcode
class InterruptSignal {
  private fired = false;
  private emitter = new EventTarget();

  fire(): void {
    this.fired = true;
    this.emitter.dispatchEvent(new Event("fire"));
  }

  isSet(): boolean {
    return this.fired;
  }

  reset(): void {
    // No epoch/generation counter needed: within one turn of the event
    // loop, nothing else can have called fire() between two statements.
    this.fired = false;
  }

  notified(): Promise<void> {
    if (this.fired) return Promise.resolve();
    return new Promise((resolve) => {
      this.emitter.addEventListener("fire", () => resolve(), { once: true });
    });
  }
}
```

### How the two primitives relate

`RuntimeTaskScope`'s `CancellationToken` and `InterruptSignal` are two
separate, differently-shaped primitives, not one wrapping the other.
`CancellationToken` comes off the shelf from `tokio-util` and shuts down
whole server tasks with built-in parent/child propagation, but it's
one-shot — once cancelled, it stays cancelled; there's no "reset." That's
exactly why the agent turn needed its own hand-rolled primitive:
`InterruptSignal`'s epoch-guarded reset lets a cancel be cleared and
re-armed for the next turn, which `CancellationToken` simply has no
concept of.

**An honest gap, not filled in:** the Phase 2 research behind this chapter
traced `InterruptSignal`'s own file thoroughly, but did not trace the
actual call sites where an Esc/Ctrl+C keypress calls `.fire()`, or where
the agent's turn loop calls `.notified()`/`.is_set()`. The doc comments
state the intent clearly enough ("eliminates spin-loops during tool
execution," a reference to "agent stream loop, tool-wait select" in the
`notified()` comment) but that's a statement of intent, not a confirmed
call path — so this chapter stops at "this is what the primitive does and
why it's shaped this way," not "here's the exact line where Esc reaches
it."

## Takeaways

- A spawned task with no owner is a task nobody can stop. `RuntimeTaskScope`
  exists purely to make sure every background task has an owner, at all
  times, until it's done.
- Cancellation in tokio (and in JS's `AbortSignal`) is cooperative: it's a
  flag a task has to notice, not a kill switch. Both models share this
  limitation.
- The deadlock `RuntimeTaskScope::shutdown` dodges — holding a lock across
  an await — is a genuinely Rust-shaped hazard. It doesn't have a JS
  analogue, because JS has no second thread to race against.
- `InterruptSignal`'s epoch counter is solving a true-concurrency problem.
  It's one of the few places in this codebase where the idiomatic TS
  translation is meaningfully *simpler* than the Rust original, and that
  gap is itself the lesson.
