# Phase 2 Analysis Notes — Chapter 3: Fan-out / Fan-in Patterns

Verified personally by re-opening every cited file at the cited lines on
2026-08-12 (same commit as the Phase 1 map, `master` @ `5ae238574`). All line
numbers below were read directly in this pass, not copied from the map
without re-check. Where my re-read matches the map exactly, I say so; where
I add detail the map didn't call out, I flag it as new.

Three genuinely distinct fan-out/fan-in patterns live in this codebase, and
the chapter's job is to make a reader reach for the *right* one, not just
know all three exist. Framing for the chapter intro: all three answer "how
do I run several things at once and then do something with all their
results," but they differ on (a) is the result-set known up front or
discovered as work runs, (b) do you want the *whole* batch's outcome or do
you want to react as each piece finishes, and (c) is the "fan-in" a single
call that blocks until done, or a long-lived loop that's woken up by events.

---

## CROSS-CHECK CORRECTION (added after Ch2's notes landed — read this before drafting)

The Chapter 2 notes pass (`notes-ch02.md` §0) traced every caller of
`run_swarm_message`/`run_swarm_task` (the function containing this
chapter's Pattern 1 `try_join_all` call) and found it is reachable **only**
from two debug-socket string commands, not from the live `swarm` tool a
production agent calls:

```rust
// crates/jcode-app-core/src/server/debug_command_exec.rs:132-139
if trimmed.starts_with("swarm_message:") { ... run_swarm_message(agent.clone(), msg).await? ... }
```
```rust
// crates/jcode-app-core/src/server/debug_jobs.rs:77-95
if trimmed.starts_with("swarm_message_async:") { ... run_swarm_message(agent.clone(), &msg).await ... }
```

The `swarm` tool's own `"message"` action (`crates/jcode-app-core/src/tool/communicate.rs:2261-2276`) does something unrelated — it routes a DM/broadcast/channel post, per its own comment: *"`message` is the general-purpose send: it routes by the fields provided... With `to_session` it acts as a DM, with `channel` it posts to that channel, and with neither it broadcasts to the sender's spawned subtree."* It never calls `run_swarm_message`.

**This does not invalidate Pattern 1 as a teaching example** — it's still real, shipped, exact code demonstrating the `try_join_all` fan-out/fan-in idiom, and worth keeping as the chapter's flagship illustration of that Rust pattern. But Phase 3 MUST label it accurately: **a debug/dev-tooling code path**, not "what happens when an agent calls the swarm tool to fan out work." Say so explicitly in the chapter text — don't let the reader infer this is the live production spawn mechanism (that's `spawn_swarm_agent`, covered in Chapter 2). A one-sentence caveat right where Pattern 1 is introduced is sufficient; it doesn't need to dominate the chapter.

---

## Pattern 1 — `try_join_all` planner fan-out (swarm.rs)

**Plain English first:** this is the "ask an LLM to break a task into 2-4
independent pieces, run all the pieces in parallel, then ask the LLM to
combine the results" pattern. It's the shape most people picture when they
hear "swarm of agents": one coordinator plans, N workers execute
concurrently, one coordinator step reassembles. The key property: the
*set* of parallel tasks is fixed before any of them start (the planner
decides it up front), and the caller needs *all* of them to finish
successfully before doing anything else — if one fails, there is no
sensible way to write the "combine" step, so the whole thing should fail
fast.

**Where:** `crates/jcode-app-core/src/server/swarm.rs`

**The exact flow, `run_swarm_message` — lines 1615-1701** (verified by
direct read, not the map's paraphrase):
1. Lines 1630-1634: build `planner_prompt` — "Break the request into 2-4
   subtasks. Return ONLY a JSON array of objects with keys: description,
   prompt, subagent_type."
2. Line 1638: `agent.run_once_capture(&planner_prompt).await?` — the
   *planner* step. Runs on the same `Arc<Mutex<Agent>>` the coordinator
   itself is using (locked for the duration of this call).
3. Line 1641: `let mut tasks = parse_swarm_tasks(&plan_text);` — parses that
   JSON into `Vec<SwarmTaskSpec>`. `SwarmTaskSpec` is defined at lines
   1703-1709 (`description: String`, `prompt: String`, `subagent_type:
   Option<String>`, `#[derive(Debug, Deserialize)]`). If parsing fails or
   yields zero tasks (lines 1642-1648), it falls back to a single-item
   `vec![SwarmTaskSpec { description: "Main task", prompt: message, ... }]`
   — i.e. a malformed plan degrades to "just do it as one task" rather than
   erroring out.
4. Lines 1657-1670: `let task_futures = tasks.iter().map(|task| { ... async
   move { run_swarm_task(agent, &description, &subagent_type, &prompt)
   .await?; ... } });` — builds an iterator of futures. **This is the
   fan-out**: nothing has run yet, this just constructs the work items.
5. **Line 1671: `let task_outputs = try_join_all(task_futures).await?;`**
   — **this is the exact citation for the flagship pattern.** Confirmed
   precisely at line 1671 (the map's own correction: the plan document's
   rough guess of "1652-1671" spans task_futures construction through this
   call — both numbers land on the same passage, 1671 is the call itself).
6. Lines 1673-1689: an *integration* step — a third `agent.run_once_capture`
   call (line 1688) that feeds every subagent's `(description, output)`
   pair back into the same locked coordinator agent to produce the final
   answer.

So the real shape is **plan → fan-out/fan-in → integrate**: three
*sequential* LLM calls, with exactly one concurrent step in the middle.
Worth being explicit in the chapter that the coordinator agent is
single-threaded through all three steps (it's an `Arc<Mutex<Agent>>` locked
for planning and again for integration) — only the *subagent* work is
concurrent.

**What each fanned-out task actually is — `run_swarm_task`, lines
1528-1613** (verified by direct read):
- Forks a brand-new `Session` (`Session::create`, line 1548) and `Agent`
  (`Agent::new_with_session`, line 1583).
- Inherits the parent's exact model/provider/auth identity (lines 1535-1546
  capture `provider_fork()`, `provider_model()`, `session_provider_key()`,
  `session_route_api_method()` from the locked parent agent; lines
  1553-1558 apply them to the child session) — a design comment at
  1554-1556 explains why: "so the forked worker keeps the same
  provider/auth route... instead of silently falling back to the config
  default."
- Strips `subagent`, `task`, `todo`, `todowrite`, `todoread` from the
  child's allowed tool set (lines 1575-1578) — a forked worker cannot
  itself spawn more swarm tasks. This is a real, load-bearing guard against
  runaway recursive fan-out from *this specific* pattern (contrast: the
  deep-mode DAG, covered elsewhere in the handbook, does allow recursive
  decomposition — that's a structurally different mechanism with its own
  cap).
- Runs exactly one prompt to completion: `worker.run_once_capture(prompt)`
  at line 1584, and returns its string output (or propagates the error,
  lines 1598-1611) — this `Result<String>` return type is exactly what
  `try_join_all` needs: `try_join_all` requires every future to yield the
  same `Result<T, E>` shape so it can either collect all the `T`s or
  short-circuit on the first `E`.

**Failure semantics (why this pattern, specifically):** `try_join_all`
short-circuits — the instant any one task future resolves `Err`, the whole
`await` returns that error immediately; the other still-running futures are
dropped (their tasks are not explicitly cancelled by anything in this
function — dropping a future that's mid-`.await` on tokio simply stops
polling it, which for an in-process async task effectively abandons it).
This is the right choice *here* specifically because the integration step
that follows has no sensible partial-success behavior to fall back to — you
can't "combine 2 of 3 planned subtask outputs" into a coherent final answer
when the plan assumed all 3 would exist.

**[RUST] plain English** (already flagged in the map, worth repeating here
verbatim since it's the core teaching point): `try_join_all` (from the
`futures` crate) is exactly `Promise.all` with early-rejection semantics —
run every future concurrently, resolve with a `Vec` of all outputs if every
one succeeds, or reject as soon as any one fails.

**TS teaching equivalent:** `Promise.all(tasks.map(spawnSubtask))` —
almost a direct 1:1 mapping. The one nuance worth calling out explicitly in
Phase 3's prose: JS's `Promise.all` also abandons the other in-flight
promises' *results* on first rejection (they keep running to completion in
the background but their resolution is ignored) — the same "fire, don't
truly cancel" semantics as the Rust dropped-future case, which is a good
one-sentence parallel to draw for readers who assume `Promise.all`
cancels its siblings (it doesn't, and neither does this).

---

## Pattern 2 — `FuturesUnordered` intra-agent batch tool (tool/batch.rs)

**Plain English first:** this is a different problem: a single agent turn
wants to run several *independent tool calls* (e.g. reading 5 files) at
once, as part of ONE tool invocation ("batch"), and wants to show live
"N of M done" progress to the UI as each one finishes — not just a single
blocking wait for all of them. The tasks here aren't LLM-planned subagents
with their own sessions; they're plain tool calls dispatched through the
same `Registry`. The defining need is *incremental* observation of
completion order, which is exactly what `try_join_all`/`Promise.all` cannot
give you (it only resolves once, with everything, in original order).

**Where:** `crates/jcode-app-core/src/tool/batch.rs`, `BatchTool::execute`
(the `Tool` trait impl), lines 218-366 in the file as it stands today —
re-read in full in this pass, matches the map's line numbers exactly.

**The exact flow (verified):**
1. Lines 218-238: parse/validate input; enforce `MAX_PARALLEL` (a `const
   MAX_PARALLEL: usize = 10` at line 10) at lines 226-231; reject
   self-nested `batch` calls at lines 234-238.
2. Lines 270-280: publish an *initial* `BusEvent::BatchProgress` — every
   subcall marked `Running`, `completed: 0` — onto the **global** `Bus`
   (`crate::bus::Bus::global().publish(...)`) *before* any concurrent work
   starts. This is the bridge to Chapter 7's bus material: progress for an
   intra-turn concurrency primitive is surfaced through the same
   process-wide broadcast channel that everything else in the UI listens
   to.
3. **Lines 282-295 — the exact `FuturesUnordered` construction, re-verified
   character-for-character against the file:**
   ```rust
   let mut stream: futures::stream::FuturesUnordered<_> = subcalls
       .iter()
       .map(|(i, tool_name, parameters)| {
           let registry = self.registry.clone();
           let i = *i;
           let tool_name = tool_name.clone();
           let parameters = parameters.clone();
           let sub_ctx = ctx.for_subcall(format!("batch-{}-{}", i + 1, tool_name.clone()));
           async move {
               let result = registry.execute(&tool_name, parameters, sub_ctx).await;
               (i, tool_name, result)
           }
       })
       .collect();
   ```
   Each future captures its own original index `i` and returns
   `(i, tool_name, result)` — this is how the code can later restore
   original request order even though the stream itself yields in
   *completion* order.
4. **Lines 300-317 — the drain loop:**
   ```rust
   while let Some((i, tool_name, result)) = stream.next().await {
       // ... publish one BatchProgress update per completion (lines 305-315)
   }
   ```
   Each iteration is one sub-call finishing (in whatever order they
   actually complete), and each publishes its own `BusEvent::BatchProgress`
   update with an incremented `completed` count and the newly-completed
   tool's name (`last_completed`) — this is the live "N of M done" signal.
5. Line 319: `results.sort_by_key(|(i, _, _)| *i)` — re-sorts back to
   original submitted order for the final formatted text output, since step
   4 drained in completion order.
6. Lines 327-347: formats every sub-call's output/error, capping each
   tool's contribution to `50_000 / num_tools` characters (line 332) so one
   giant sub-result can't crowd out the others in the combined text blob
   handed back to the LLM.

**Why this pattern specifically, not `try_join_all`:** the defining
difference from Pattern 1 is *when you get to react*. `try_join_all` gives
you nothing until everything is done. `FuturesUnordered` + a `while let
Some(...) = stream.next().await` loop gives you one item at a time, in
whatever order they actually finish, which is exactly what's needed to
publish incremental progress events. The final `results.sort_by_key`
step is a nice concrete detail for the chapter: getting completion-order
efficiency *and* deterministic output order isn't a contradiction, it just
costs one extra sort at the end.

**[RUST] plain English** (from the map, re-confirmed accurate against the
re-read code): `FuturesUnordered` is a collection of in-flight futures
polled as a group; `.next().await` yields whichever one finishes next. It
is the right tool specifically when you want to react incrementally as each
parallel job finishes, rather than only once the whole batch is done.

**TS teaching equivalent:** there's no single built-in with this exact
shape. The two honest options for Phase 3 to sketch: (a) manually racing an
array of promises in a loop — keep a `Map` of pending promises each wrapped
to resolve `(index, result)`, `await Promise.race([...pending.values()])`,
remove the winner, repeat until empty; or (b) a small async generator that
`yield`s each result as it settles. Either is close in spirit to
`FuturesUnordered`; note for Phase 3 that naive `Promise.race` re-racing in
a loop is O(n) per completion unless you're careful (each `race` call
re-wraps every remaining promise), so a hand-rolled counter/resolver
pattern is the more honest "idiomatic" analogy, not literally
`Promise.race` in a loop.

---

## Pattern 3 — event-driven fan-in via `broadcast` + `tokio::select!` (comm_await.rs)

**Plain English first:** this is a "wait until some/all of a set of already
-running agents reach a target state, but time out if they don't, and don't
poll in a tight loop burning CPU" pattern. Unlike Pattern 1, the "workers"
here are not started by this function — they're already-running swarm
members doing their own independent work; this pattern only *waits* on
them. Unlike Pattern 2, there's no bounded fixed batch of futures to
collect — it's an indefinite wait against live, externally-mutated shared
state, woken up by a stream of events rather than driven to completion by
directly awaiting the workers' futures.

**Where:** `crates/jcode-app-core/src/server/comm_await.rs`.

**The exact flow, `spawn_or_resume_await_members` — lines 209-303**
(re-read in full, matches the map's line numbers exactly, including the
map's own correction of the plan document's rough `222-275` guess):
- Line 223: `tokio::spawn(async move { ... })` — the whole watcher runs as
  a detached background task, not something the caller awaits directly (the
  caller gets a handle to poll/attach to later via
  `await_members_runtime`, not covered further here — out of scope for this
  chapter).
- Line 224: `let mut event_rx = swarm_event_tx.subscribe();` — subscribes
  to the swarm's `broadcast::Sender<SwarmEvent>`.
- Lines 227-301: a `loop`. On every iteration:
  - Lines 228-236: re-reads every watched member's current status directly
    from shared state via `awaited_member_statuses(...)` — **not** from the
    event stream. This is the single most important design point for the
    chapter: the event is only a wake-up nudge; the *source of truth* is
    always the live registry, read fresh every loop iteration.
  - Lines 238-255: if the wait's mode (`"any"` vs `"all"`, decided by
    `mode_satisfied`, lines 102-107 in the same file) is already satisfied,
    or there's nothing left to watch, call `finalize_await` and return —
    done.
  - Lines 257-269: for *blocking* (non-background) waits, if every
    socket-side waiter has disconnected, give up and clean up (no point
    watching for a client that's gone).
  - **Lines 271-300 — the `tokio::select!` fan-in itself**, racing exactly
    two branches:
    1. `_ = tokio::time::sleep_until(deadline) => { ... }` (line 272) — the
       timeout branch: on firing, calls `finalize_await(..., false, ...,
       timeout_summary(...))` and returns.
    2. `event = event_rx.recv() => { ... }` (line 277) — a new
       `SwarmEvent` arrived. Filtered to the same `swarm_id` (lines
       280-282, `continue`s past events from a different swarm sharing the
       same broadcast channel). On `Err(RecvError::Lagged(n))` (lines
       284-292), the receiver missed `n` buffered events because it fell
       behind — the code explicitly does **not** treat this as fatal, just
       logs and `continue`s the loop (which re-reads shared state at the
       top regardless). On `Err(RecvError::Closed)` (lines 294-297,
       meaning every sender was dropped), the watcher gives up and cleans
       up.

**Why "lossy broadcast + always-refresh-from-shared-state" is *safe*
here** — this is the single best design-rationale quote for the chapter,
verified present verbatim at lines 284-287:
> "Dropped events are recoverable: the loop re-reads member statuses from
> shared state at the top, so just keep watching instead of orphaning the
> wait."

The event stream's only job is to wake the loop up sooner than the next
timeout tick; a missed event costs at most a slightly stale wakeup
(bounded by however long until the *next* real event or the timeout),
never an incorrect final result, because the actual satisfied/not-satisfied
check always re-reads the authoritative shared registry, never trusts the
event payload itself.

**Contrast with `try_join_all` explicitly worth drawing for the chapter,**
already flagged in the map (`completion_mode`/`mode_satisfied`, lines
95-107 of comm_await.rs, re-confirmed): the `"any"`/`"all"` choice here is
semantically `Promise.race` vs. `Promise.all`, but implemented as a
polling/event-driven loop against externally-owned state, not a single
combinator call over futures you started yourself. You reach for this
pattern instead of `try_join_all` specifically when the things you're
waiting on are *not futures you hold* — they're independent, already
-running agents whose progress lives in shared mutable state, and you need
a deadline plus a background/foreground duality (blocking clients vs.
persisted background waits that survive a disconnect).

**Bridge to Chapter 7:** `finalize_await` (lines 182-207, re-read in
full) — when a wait was started `background: true` **and**
`notify`/`wake` was requested (line 194), it additionally publishes a
`BusEvent::SwarmAwaitCompleted` onto the **global** `jcode-base::bus::Bus`
(line 196, `Bus::global().publish(...)`) — this is the exact hand-off point
between the per-swarm `broadcast::Sender<SwarmEvent>` used for this
fan-in loop and the separate, global bus covered in Chapter 7.

**[RUST] plain English** (from the map, re-confirmed accurate):
`broadcast::Receiver<T>::recv()` returns `Ok(event)`, `Err(Lagged(n))` if
the ring buffer overwrote `n` unread messages before this receiver caught
up, or `Err(Closed)` once every sender is dropped. Closest teaching analogy:
an `EventEmitter` with a bounded ring buffer that can silently drop old
events under backpressure — except Rust's broadcast channel tells the
receiver explicitly via `Lagged(n)` rather than silently and invisibly
dropping.

**TS teaching equivalent:** an async generator or `EventEmitter` around a
`Promise.race([sleep(deadline), nextEvent()])` loop, where `nextEvent()`
resolves off a bounded queue and the loop body re-derives "are we done yet"
from a shared status map on every iteration rather than trusting the event
payload — Phase 3 should make the "always re-derive from source-of-truth
state on wake, never trust the event payload as authoritative" property
an explicit, named idiom in the TS sketch, since it's the load-bearing
correctness property of the whole pattern.

---

## Side-by-side decision table (for the chapter's own "when would you reach
for each one" framing, which the plan explicitly asks for)

| | Pattern 1: `try_join_all` | Pattern 2: `FuturesUnordered` | Pattern 3: `broadcast` + `select!` |
|---|---|---|---|
| What you're waiting on | Futures you just started, whose *set* is fixed up front (a parsed LLM plan) | Futures you just started, whose count is small/bounded (≤10, `MAX_PARALLEL`) | Already-running, independently-owned agents; you didn't start their work |
| When you find out about progress | Only once, all-at-once, at the end | Incrementally, one at a time, in completion order | Incrementally, via wake-ups, but re-verified against shared state each time |
| Failure behavior | Short-circuits on first error (`Err`) | Every sub-call's `Result` is collected individually; no short-circuit — a batch reports N succeeded / M failed | No inherent "failure" — it's a status-matching wait with a timeout, not a `Result`-producing combinator |
| Needs a deadline? | No | No | Yes — this is the only one of the three built around a timeout |
| Source citation | `swarm.rs:1671` (call), `1657-1670` (fan-out build) | `tool/batch.rs:282-295` (build), `300-317` (drain) | `comm_await.rs:271-300` (`select!`), `209-303` (whole function) |
| TS analogy | `Promise.all` | Hand-rolled incremental-drain (race-in-a-loop or async generator) | `EventEmitter`/async-iterator + timeout race, re-deriving state each wake |

---

## Open questions / things I could NOT verify (flag explicitly, do not
paper over)

- I did not verify what happens to the *other* still-running task futures
  in Pattern 1 when `try_join_all` short-circuits on an error — I stated
  above (based on general tokio/futures semantics: dropping a future stops
  polling it) that they are abandoned rather than explicitly cancelled, but
  I did not find an explicit doc comment in `swarm.rs` confirming this is
  understood/intended behavior for `run_swarm_message` specifically (e.g.
  whether an already-spawned child `Session`'s on-disk state or running
  `Agent` task is cleaned up). This is a reasonable inference from Rust's
  drop semantics, not a directly-cited claim — Phase 3 should phrase it as
  "the future is dropped, which stops polling it" (a documented language
  property) rather than asserting jcode has explicit cancellation logic
  here, since I did not find one.
- I did not trace `await_members_runtime`'s internals (e.g.
  `retain_open_waiters`, `clear_active`) beyond what's visible in the
  `spawn_or_resume_await_members` call sites shown above — those are
  out-of-scope bookkeeping for this chapter's fan-in pattern itself, not a
  gap in the core pattern's citation.
