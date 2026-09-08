# Phase 1 Explore — File/Abstraction Map

Source of truth for the jcode swarm-architecture handbook. Every entry below
was verified by opening the cited file at the cited lines on 2026-08-12
against the current `master` branch (HEAD `5ae238574`). Line numbers are
exact as of that commit; they will drift as the code changes, so re-check
before citing in a much later session.

Status legend: all files listed in the task brief were confirmed to exist
(`wc -l` succeeded on all of them) unless flagged otherwise under
"Broken/Missing" at the end.

Rust-concept flags use the convention: **[RUST]** *plain English: ...*

## How this map is organized (read this first)

Sections below are grouped by chapter, but were written incrementally as
files were explored, so **physical order in this document is not strictly
1-8** — a chapter's material may appear in more than one place (e.g.
Chapter 3 has three separate subsections, Chapter 7 has three). Use this
table of contents to navigate; each row is a real `##`/`###` heading below.

| Chapter | Section heading (physical location, top to bottom) |
| --- | --- |
| 1 | "Chapter 1 — Why parallel agents, and why this codebase (framing)" — stub only, no dedicated file |
| 3, 7 | "Chapter 3 — Fan-out / fan-in patterns, Chapter 7 — Communication topology" → `swarm.rs` |
| 2 | "Chapter 2 — Anatomy of a spawn (continued)" → `comm_session.rs` |
| 3 | "Chapter 3 — Fan-out / fan-in patterns (continued)" → `comm_await.rs` |
| 3 | "Chapter 3 — Fan-out / fan-in patterns (continued): intra-agent batch tool" → `tool/batch.rs` |
| 7 | "Chapter 7 — Communication topology (continued): the global event bus" → `jcode-base/src/bus.rs` |
| 7 | "Chapter 7 — Communication topology (continued): `jcode-swarm-core`'s non-DAG contents" → `jcode-swarm-core/src/lib.rs` |
| 6 | "Chapter 6 — The task DAG (deep mode)" → CRITICAL FINDING + `jcode-plan/src/dag/*`, `bridge.rs`, `lib.rs` |
| 8 | "Chapter 8 — Design-pattern glossary & further reading: the two design docs" → `SWARM_ARCHITECTURE.md`, `SWARM_TASK_GRAPH.md` |
| 4 | "Chapter 4 — Structured concurrency & cancellation" → `runtime.rs`, `jcode-agent-runtime/src/lib.rs` |
| 5 | "Chapter 5 — Shared state without locks-as-a-crutch" → cross-cutting synthesis, `state.rs` |

The single most important thing to read before writing Chapter 6: the
**"CRITICAL FINDING"** subsection under the Chapter 6 heading. It corrects
a wrong assumption in the original plan about which crate owns the DAG
engine.

The final section at the bottom of this document ("Verification /
broken-links checklist") lists every file checked and confirms none are
missing.

---

## Chapter 1 — Why parallel agents, and why this codebase (framing)

No single source file "owns" this chapter; it's synthesized from the overall
shape of files below plus `docs/SWARM_ARCHITECTURE.md` §1-2 (see the doc-map
section at the bottom). Vocabulary (agent/session, coordinator, swarm,
worktree manager) should be pulled from `docs/SWARM_ARCHITECTURE.md`.

---

## Chapter 3 — Fan-out / fan-in patterns, Chapter 7 — Communication topology

### `crates/jcode-app-core/src/server/swarm.rs` (3170 lines total; real code ends ~1725, rest is `#[cfg(test)] mod tests` at line 1727-3170)

Confirmed present, 3170 lines. Note for later phases: **do not cite anything
past line 1726 as production behavior** — it's all test code (`mod tests`
starts at line 1727, guarded by `#[cfg(test)]` at line 1726).

Imports worth noting for framing (lines 1-20): `use futures::future::try_join_all;`
(line 9), `use jcode_swarm_core::{completion_notification_message,
normalize_completion_report, truncate_detail};` (lines 10-12) — confirms
swarm.rs *consumes* jcode-swarm-core helpers rather than owning them.

- **`MAX_SWARM_MEMBERS`** — line 32, re-exported from `jcode_swarm_core`
  (`pub(super) use jcode_swarm_core::MAX_SWARM_MEMBERS;`), NOT defined here.
  Doc comment (lines 26-31) — good Chapter 2/7 quote on the member cap and
  spawn-tree shape:
  > "Maximum number of live members (agents) in a single swarm... Normal and
  > light swarms are root-only, one-level fan-out. Deep-swarm roots may
  > create recursive trees with no depth limit, but both the configurable
  > live-worker budget and this absolute cap still apply."
  Chapter: 2 (mode-gated spawning: ad hoc/light/deep), 6 (task DAG deep
  mode), 7.

- **`swarm_ancestors`** — lines 41-60. One sentence: walks the
  `report_back_to_session_id` chain upward from a session to reconstruct the
  spawn tree without a separate stored parent field, cycle-guarded with a
  visited set.
  Doc comment (lines 34-40) is the key design-rationale quote for Chapter 2:
  > "The spawner/parent edge is encoded by `report_back_to_session_id`: a
  > child spawned by `P` reports back to `P`. Walking that chain
  > reconstructs the spawn tree without persisting a separate parent field."
  Chapter: 2 (spawn ancestry — this is exactly the mechanism named in the
  plan outline).

- **`swarm_spawn_depth`** — lines 68-71, `#[cfg(test)]`-only now (doc
  comment lines 62-67 explicitly says production code no longer enforces or
  consults a depth cap; kept only because spawn-tree tests assert depth).
  **Flag: do not describe a depth limit as currently enforced** — the doc
  comment is explicit that it isn't ("the spawn tree no longer enforces a
  depth cap, so production code does not consult depth").

- **`swarm_is_self_or_ancestor`** — lines 76-85. One sentence: authorization
  check — true if `ancestor` is `session_id` itself or any transitive
  spawner, used to decide whether a requester may stop/control a target
  (an agent owns its entire spawned subtree). Chapter: 2, 7.

- **Status-broadcast debounce machinery** — lines 87-231 roughly:
  `PendingSwarmStatusBroadcast` struct (102-106), `pending_swarm_status_broadcasts()`
  (108-113, a process-global `OnceLock<StdMutex<HashMap<...>>>`),
  `swarm_status_debounce_member_threshold`/`swarm_status_debounce_ms`
  (115-141, env-var-configurable), constants at 87-101 including doc comment
  at lines 94-100 explaining *why* terminal members are dropped from live
  broadcasts after a retention window:
  > "re-sending hundreds of long-finished members to every attached client
  > on every status change dominates broadcast payloads (measured ~240 KB of
  > member JSON resident per client with ~700 mostly-stopped members)."
  **[RUST]** *plain English: `OnceLock` is a cell that can be initialized
  exactly once and then read many times without further locking — used here
  to create a lazily-initialized global/static piece of shared state (a
  singleton), similar to a module-level `let cache = null; function get() {
  if (!cache) cache = new Map(); return cache; }` pattern in JS.*
  Chapter: 5 (shared state) — a concrete case of the codebase choosing
  debounced eventual-consistency broadcast over locking harder.

- **`swarm_idle_worker_reap_after`** (266-272) / **`idle_spawned_worker_reap_candidates`**
  (279-291) — reaper for idle spawned workers. Doc comment (260-263)
  explains the *why*:
  > "Spawned workers... rely on their coordinator calling `cleanup`, but ad
  > hoc spawns and interrupted plans leave them behind, where each idle
  > client holds ~80-150 MB indefinitely. The reaper is the backstop..."
  Chapter: 2 (spawn lifecycle/cleanup), 4 (cancellation/cleanup discipline).

- **`DeadMemberSalvage` struct** — lines 294-300 (`requeued_task_ids`,
  `failed_task_ids`), with `is_empty`/`describe` methods (302-330).
  **`salvage_plan_assignments_of`** — lines 344-390 (sync, mutates a
  `VersionedPlan` in place). **`salvage_assignments_of_dead_member`** —
  lines 396-449 (async wrapper: salvage, persist, broadcast plan, notify
  coordinator). **`notify_coordinator_of_salvage`** — lines 453-489.
  One sentence: when a worker dies mid-task, its non-terminal plan items are
  automatically requeued (or, past `MAX_DEAD_ASSIGNEE_RECLAIMS`, marked
  failed) so a driving `run_plan` doesn't stall silently on a corpse's
  assignment. Doc comment (333-343) has good Chapter 3/7 rationale on why
  this exists eagerly rather than relying only on assign-time reclaim.
  Chapter: 3 (fan-in failure handling — what happens when a fanned-out
  child dies), 7.

- **`touch_swarm_task_progress`** — lines 495-553ish (signature 495-500,
  `#[expect(clippy::too_many_arguments...)]` at 491-494 — note: the doc
  attribute is a real, load-bearing code comment about *why* the function
  has many params, safe to quote). **`refresh_swarm_task_staleness`** —
  554-672ish. Heartbeat/staleness sweep for plan-driven task assignment.
  Chapter: 3, 6.

- **`broadcast_swarm_status`** (742-812) — debounced fan-out of live member
  status to all swarm participants; when member count is below
  `swarm_status_debounce_member_threshold()` broadcasts immediately (line
  758-761), otherwise schedules a coalesced `tokio::spawn`'d flush loop
  (783-811) so N rapid status changes collapse into one broadcast.
  **`broadcast_swarm_status_now`** — lines 685-741 (the actual send).
  Chapter: 7 (communication topology — status snapshot vs. full broadcast).

- **`broadcast_swarm_plan`** (818-834, thin wrapper) /
  **`broadcast_swarm_plan_with_previous`** (836-920) — the authoritative
  plan-graph fan-out. Doc comment (814-817):
  > "Plan snapshots are sent to explicit plan participants. If a plan has no
  > participants yet, fall back to all current swarm members."
  Delivery loop at lines 899-908 sends `ServerEvent::SwarmPlan` to each
  participant's own `member.event_tx` (a per-member channel, not the global
  bus) — i.e. plan/status fan-out is targeted unicast-per-member, not a
  single broadcast channel. Chapter: 7 — good contrast case against
  `jcode-base/src/bus.rs`'s single global broadcast.

- **`send_swarm_plan_to_session`** — lines 926-970. Doc comment (922-925)
  explains *why* it exists distinctly from broadcast: reconnecting clients
  need an immediate snapshot rather than waiting for the next mutation.
  Chapter: 7.

- **`rename_plan_participant`** (972-982), **`remove_plan_participant`**
  (984-993), **`remove_session_from_swarm`** (995-1219, the largest single
  function in the file — membership teardown/bookkeeping) — member registry
  mutation. Chapter: 2, 7.

- **`set_member_task_label`** (1219-1233ish), **`record_swarm_event`**
  (1233-1259), **`record_swarm_event_for_session`** (1259-1291) — event
  history bookkeeping (feeds `event_history: Arc<RwLock<VecDeque<SwarmEvent>>>`
  seen in `runtime.rs:107`). Chapter: 7.

- **`update_member_status`** (1291-1313) → **`update_member_status_with_report`**
  (1319-1343) → **`update_member_status_with_report_tldr`** (1349-1527ish):
  a three-layer delegation chain, each adding one more optional parameter
  (plain status → +completion_report → +tldr). One sentence: the single
  choke point that updates a member's status in the registry and triggers
  the corresponding notifications/broadcasts. Both wrapper layers carry
  `#[expect(clippy::too_many_arguments, reason = "...")]` attributes (lines
  1315-1318, 1345-1348) documenting *why* the signature is wide — worth
  quoting as evidence the "too many args" shape is a deliberate, acknowledged
  tradeoff, not an oversight. Chapter: 3, 7.

- **`run_swarm_task`** — lines 1528-1613. One sentence: forks a brand-new
  child `Session`+`Agent` (via `Session::create`, line 1548, and
  `Agent::new_with_session`, line 1583) that inherits the parent's provider
  auth/model/working-dir, runs one prompt to completion via
  `worker.run_once_capture(prompt)` (line 1584), and returns its output —
  this is the unit of work `try_join_all` fans out over. Also strips
  swarm/task/todo tools from the child's allowed set (lines 1575-1581) so a
  forked worker can't recursively spawn more swarm tasks itself.
  **[RUST]** *plain English: `Arc<Mutex<Agent>>` (parameter type, line
  1529) is a "shared, thread-safe handle to one mutable value" — `Arc`
  (atomic reference count) lets many owners hold the same object, `Mutex`
  ensures only one at a time can mutate it. This is the standard
  Rust/tokio pattern for sharing one live object across concurrently
  spawned tasks; the closest TS analogue is just holding one shared object
  reference plus a manual async lock/queue if you need mutual exclusion,
  since JS is single-threaded by default and doesn't need `Arc`.*
  Chapter: 2 (this literally is "anatomy of a spawn" at the swarm-task
  level, complementary to `comm_session.rs`'s `spawn_swarm_agent`), 3.

- **`run_swarm_message`** — lines 1615-1701. **This is the
  planner → fan-out → fan-in flow named in the plan.** Structure, all
  verified by reading lines 1615-1701 directly:
  1. Lines 1630-1634: builds a `planner_prompt` asking the agent to "Break
     the request into 2-4 subtasks" and return JSON.
  2. Line 1638: `agent.run_once_capture(&planner_prompt).await?` — the
     *planner* step, run on the same locked agent.
  3. Line 1641: `parse_swarm_tasks(&plan_text)` parses that JSON into
     `Vec<SwarmTaskSpec>` (falls back to a single "Main task" spec if
     parsing/empty, lines 1642-1648).
  4. Lines 1657-1670: builds `task_futures`, an iterator of `async move`
     blocks, each calling `run_swarm_task(...)` — this is the **fan-out**.
  5. Line 1671: `let task_outputs = try_join_all(task_futures).await?;` —
     the **fan-in**: runs every task future concurrently and either
     collects all `Ok` outputs or short-circuits on the first `Err`.
  6. Lines 1673-1689: an *integration* step — a third agent call
     (`agent.run_once_capture(&integration_prompt)`, line 1688) that feeds
     all subagent outputs back to the (locked) parent agent to produce the
     final answer. So the full shape is **plan → fan-out/fan-in → integrate**,
     three sequential agent calls around one concurrent middle step.
  **[RUST]** *plain English: `try_join_all` (from the `futures` crate) is
  exactly `Promise.all` with early-rejection semantics — run every future
  concurrently, resolve with a `Vec` of all outputs if every one succeeds,
  or reject as soon as any one fails. It is the fan-in primitive for this
  pattern.*
  Chapter: 3 (this is the flagship `try_join_all` example the plan calls
  out — verified location, actually **line 1671**, not 1652-1671 as the
  plan's rough note guessed; 1652-1671 spans the `task_futures` construction
  through the `try_join_all` call, so both numbers point at the same
  passage, just different start points).

- **`SwarmTaskSpec` struct** — lines 1703-1709 (`description`, `prompt`,
  `subagent_type: Option<String>`, `#[derive(Debug, Deserialize)]`) — the
  planner's structured-output contract.

- **`parse_swarm_tasks`** — lines 1711-~1725. Parses the planner's JSON
  array, tolerant of surrounding prose (falls back to slicing between the
  first `[` and last `]`, lines 1716-1717+).

---

## Chapter 2 — Anatomy of a spawn (continued)

### `crates/jcode-app-core/src/server/comm_session.rs` (1434 lines total)

Confirmed present, 1434 lines. Tests start at `#[cfg(test)]` line 1432 (just
2 lines, effectively no test module body shown but marks end of real code).

- **`SwarmSpawnMode` enum** — NOT defined in this file; it lives in
  `crates/jcode-config-types/src/lib.rs:646-657`. One sentence: the
  mode-gate for spawning named directly in the plan outline (`Visible`,
  `Headless`, `Inline` [default, line 653 `#[default]`], `Auto`). Doc
  comments on each variant (644-656) are worth quoting verbatim for Chapter
  2 since they explain the actual behavioral difference:
  > `Visible`: "Open a visible/headed terminal window. This was the
  > historical default." `Headless`: "Create the worker in-process without
  > opening a terminal window." `Inline` (default): "Like headless (no
  > terminal window), but the coordinator renders a live inline gallery
  > viewport of each worker's streaming output." `Auto`: "Try visible first
  > and fall back to headless if a window cannot be opened."
  Chapter: 2 (this **is** the "mode-gated spawning" the plan asks for —
  note it's about visible-window vs. headless/inline *rendering*, not the
  ad hoc/light/deep task-DAG modes, which are a separate axis owned by
  `jcode-plan`'s `Mode` enum — see Chapter 6 section below. Don't conflate
  the two "mode" concepts in the handbook.).

- **`spawn_swarm_agent`** — lines 557-827. **This is the function named in
  the plan.** One sentence: resolves working dir/model/provider identity for
  a new swarm member, attempts a visible (headed terminal) spawn per
  `resolved_spawn_mode`, falls back to a headless session
  (`create_headless_session`, called at line 662) if visible spawn fails or
  wasn't requested, registers the new member in the swarm registry and
  broadcasts the plan, then — **only when falling back to headless AND an
  initial message was given** (`is_headless_fallback && startup_message`,
  lines 746-748) — fires a **fire-and-forget `tokio::spawn`** (lines
  772-822) that runs the initial message on the new child agent and updates
  its status to `ready`/`failed` on completion.
  **Correction to plan's rough citation**: the plan's note says
  "`comm_session.rs:770-816` — fire-and-forget `tokio::spawn`"; the actual
  `tokio::spawn(async move { ... })` block is **lines 772-822** (the
  `tokio::spawn(` call itself is at line 772, closing at 822); 770-816 is
  inside that same block (the setup before/inside it), so the plan's number
  isn't wrong so much as imprecise — cite **772-822** for the spawn block
  itself.
  Nuance worth flagging for Chapter 2: this fire-and-forget spawn is
  **conditional** — it does NOT fire for visible spawns or for headless
  spawns with no initial message. Those paths return synchronously with
  just the new session id (line 826 `Ok(new_session_id)`) and whatever
  processes the first message happens elsewhere (client-driven).
  **[RUST]** *plain English: `tokio::spawn(async move { ... })` (line 772)
  starts a new concurrently-running task and immediately returns a
  `JoinHandle` — here the handle is dropped (not stored/awaited), so this
  is "fire-and-forget": the parent function returns `Ok(new_session_id)`
  without waiting for that inner task to finish. Equivalent TS idiom:
  calling an `async function` without `await`ing it (a "floating promise"),
  which is normally a lint error in TS/JS because it's easy to lose errors
  silently — worth calling out as a real tradeoff of this pattern in
  Chapter 2/4 (errors from this task are only visible via the
  `update_member_status_with_report` call inside it, not via any caller
  seeing a `Result`).*
  Also worth noting inside `spawn_swarm_agent`: line 736,
  `set_member_task_label` (labels the worker for UI display); lines
  698-706, plan participant registration guarded so it only adds
  participants to plans that already have items/participants (empty plans
  aren't touched); line 744 `persist_swarm_state_for` — every spawn
  persists state to disk before the fire-and-forget task even starts.

- **`handle_comm_spawn`** — lines 830-957ish, the RPC-facing wrapper that
  calls `spawn_swarm_agent` and formats a protocol response. Chapter: 2, 7.

- **`resolve_coordinator_spawn_identity`** — lines 233-293ish,
  **`CoordinatorSpawnIdentity`** struct (210-220), **`SwarmSpawnSelection`**
  struct (221-232) — model/provider-key/route inheritance logic for spawned
  children (referenced in Chapter 2 as "inherits the parent's exact auth
  identity", matching the same idea seen in `swarm.rs`'s `run_swarm_task`
  at lines 1554-1558).

- **`create_visible_spawn_session`** (67-106), **`resolve_spawn_working_dir`**
  (107-144), **`spawn_visible_session_window_with_context`** (145-174),
  **`prepare_visible_spawn_session`** (429-478), **`register_visible_spawned_member`**
  (479-556) — the visible-window spawn machinery `spawn_swarm_agent` calls
  into. Not read line-by-line; flagged here so Phase 2 agents know where to
  look if Chapter 2 needs more visible-spawn detail than the summary above.

- **`handle_comm_stop`** — lines 987-1162ish, and **`swarm_stop_allowed_by_owner`**
  (1163-1170), **`resolve_stop_target_session`** (1171-1225) — implements
  the "an agent owns its entire spawned subtree" authorization check from
  `swarm.rs::swarm_is_self_or_ancestor` (lines 76-85) at the RPC layer.
  Chapter: 2, 4 (cancellation/stop propagation down a spawn subtree).

- **`ensure_spawn_coordinator_swarm`** — lines 1234-1431ish. Not read in
  detail; likely relevant to Chapter 2 (how a session becomes a
  "coordinator" the first time it spawns).

---

## Chapter 3 — Fan-out / fan-in patterns (continued)

### `crates/jcode-app-core/src/server/comm_await.rs` (680 lines total)

Confirmed present, 680 lines.

- **`awaited_member_statuses`** — lines 13-68. One sentence: builds the list
  of member ids to watch (either an explicit `requested_ids` list, or every
  swarm member other than the caller, lines 21-39) and, for each, computes a
  `done: bool` flag against the caller's `target_status` list (with an
  "unknown status counts as done if we're waiting for stopped/completed"
  special case, lines 55-58). Chapter: 3, 7.

- **`completion_mode`** (95-100) / **`mode_satisfied`** (102-107) /
  **`mode_summary`** (109-126): the wait has two modes — `"any"` (done when
  *any* watched member reaches target status,
  `member_statuses.iter().any(...)`, line 104) or `"all"` (default, `.iter().all(...)`,
  line 105). This is the semantic equivalent of choosing between
  `Promise.race` and `Promise.all` for the fan-in, but implemented as a
  polling/event-driven loop rather than a single combinator call — worth
  drawing that contrast explicitly in Chapter 3 alongside `try_join_all`.

- **`spawn_or_resume_await_members`** — lines 209-303. **This is the
  fan-in-via-broadcast function named in the plan** (plan cited
  `comm_await.rs:222-275`; actual content: function spans 209-303, the
  `tokio::select!` fan-in loop is lines 271-300 inside a `tokio::spawn`
  block that starts at line 223). One sentence: spawns a background task
  that subscribes to the swarm's `broadcast::Sender<SwarmEvent>` (line 224,
  `swarm_event_tx.subscribe()`) and loops, on each relevant event
  re-checking every watched member's status against the target, exiting via
  `finalize_await` when the mode is satisfied, the deadline passes, or (for
  non-background/blocking waits) every socket-side waiter has disconnected
  (lines 263-269).
  The `tokio::select!` (lines 271-300) races exactly two things:
  1. `tokio::time::sleep_until(deadline)` (line 272) — timeout branch, calls
     `finalize_await(..., false, ...)` with a `timeout_summary`.
  2. `event_rx.recv()` (line 277) — a new `SwarmEvent` arrived; filtered to
     the same `swarm_id` (lines 280-282), and on `RecvError::Lagged(n)`
     (broadcast channel overflow — the receiver missed `n` messages) the
     code deliberately does **not** treat that as fatal: comment (lines
     284-287) explains why:
     > "Dropped events are recoverable: the loop re-reads member statuses
     > from shared state at the top, so just keep watching instead of
     > orphaning the wait."
     This is a good concrete "why broadcast + polling is safe here" note
     for Chapter 3 — the event stream is just a wake-up nudge, not the
     source of truth (shared state is), so a missed event only costs a
     slightly stale wakeup, never an incorrect result.
  **[RUST]** *plain English: `broadcast::Receiver<T>::recv()` (line 277)
  can return `Ok(event)`, or `Err(Lagged(n))` if the receiver fell behind
  the ring buffer and missed `n` messages (the sender doesn't block for
  slow receivers — it overwrites old ones), or `Err(Closed)` once every
  sender has been dropped. There's no direct Node/browser built-in with
  this exact "lossy, capacity-bounded pub/sub with an explicit lag signal"
  shape — the closest teaching analogy is an `EventEmitter` where a slow
  consumer using a bounded ring buffer could silently drop events, except
  Rust's broadcast channel tells the receiver explicitly via `Lagged(n)`
  rather than silently dropping.*
  Chapter: 3 (event-driven fan-in via broadcast + `tokio::select!`, exactly
  as named in the plan), 4 (deadline/cancellation shape), 7.

- **`finalize_await`** — lines 182-207. Persists the final response (line
  192), and — only if the wait was started as `background` **and**
  `notify`/`wake` was requested (line 194) — publishes a
  `BusEvent::SwarmAwaitCompleted` onto the **global** `Bus` (line 196,
  `Bus::global().publish(...)`) — i.e. this is where the per-swarm
  `broadcast::Sender<SwarmEvent>` fan-in hands off to the separate global
  `jcode-base::bus::Bus` for cross-cutting notification delivery. Good
  bridge citation between Chapter 3 and Chapter 7's `bus.rs` section.

- **`CommAwaitMembersContext` struct** — lines 305-311 (borrowed-reference
  bundle for the RPC handler). **`handle_comm_await_members`** — lines
  317-542ish (not read in full; the public RPC entry point that validates
  the request and calls into `spawn_or_resume_await_members` or serves an
  already-persisted result). **`resume_background_awaits`** — lines
  608-680ish, restarts background waits after a server restart (persisted
  state resumption).

---

## Chapter 3 — Fan-out / fan-in patterns (continued): intra-agent batch tool

### `crates/jcode-app-core/src/tool/batch.rs` (371 lines total — full file read)

Confirmed present, 371 lines. Test module at 369-371 (`#[path =
"batch_tests.rs"] mod batch_tests;` — the actual test bodies live in a
sibling file `batch_tests.rs`, not read here).

- **`MAX_PARALLEL`** — line 10, `const MAX_PARALLEL: usize = 10` — hard cap
  on concurrent sub-tool-calls in one `batch` invocation, enforced at lines
  226-231.

- **`BatchTool` struct** — lines 96-98 (`registry: Registry`), constructed
  via `BatchTool::new` (100-103).

- **`BatchInput`/`ToolCallInput` structs** — lines 106-117, plus
  `normalize_batch_input` (128-202, doc comment 128-131) which forgives
  common LLM-generated shape mistakes (wrong key names, un-nested
  parameters) before deserializing — not concurrency-relevant but worth
  knowing it's there so a handbook snippet doesn't accidentally quote the
  normalization code as "the concurrency logic."

- **`Tool for BatchTool::execute`** — lines 218-366. **This is the
  `FuturesUnordered` concurrent tool-execution function named in the
  plan.** Verified structure:
  1. Lines 218-238: parse/validate input, enforce `MAX_PARALLEL`, reject
     nested `batch` calls (self-recursion guard, lines 234-238).
  2. Lines 270-280: publishes an initial `BusEvent::BatchProgress` (all
     subcalls `Running`, `completed: 0`) onto the global `Bus` *before* any
     work starts — this is the "progress published to `jcode-base/src/bus.rs`"
     the plan refers to.
  3. Lines 282-295: **`let mut stream: futures::stream::FuturesUnordered<_> =
     subcalls.iter().map(|...| async move { ... }).collect();`** — this is
     the exact citation for "intra-agent concurrency via
     `futures::stream::FuturesUnordered`" (plan's rough note said
     `279-291`; the `FuturesUnordered` construction itself is precisely
     **282-295**). Each future calls `registry.execute(&tool_name,
     parameters, sub_ctx).await` (line 291) and returns `(i, tool_name,
     result)` so original order can be restored later.
  4. Lines 300-317: **`while let Some((i, tool_name, result)) =
     stream.next().await { ... }`** — drains the `FuturesUnordered` in
     completion order (not submission order), publishing one
     `BusEvent::BatchProgress` update per completed sub-call (lines
     305-315) — this is the live "N of M done" progress mechanism.
  5. Line 319: `results.sort_by_key(|(i, _, _)| *i)` — re-sorts back to
     original request order for the final formatted output, since
     `FuturesUnordered` yields in completion order not input order.
  6. Lines 327-347: formats every sub-call's output/error into one text
     blob, per-tool output capped at `50_000 / num_tools` chars (line 332)
     so one huge sub-result can't crowd out the others.
  **[RUST]** *plain English: `FuturesUnordered` is a collection of
  in-flight futures that you poll as a group; each call to `.next().await`
  gives you whichever one finishes next, in completion order — this is
  different from `try_join_all`/`Promise.all` (which waits for ALL of them
  and returns everything at once in the original order). It's the right
  tool when you want to react incrementally as each parallel job finishes
  (e.g., to publish a progress event per completion, as this code does)
  rather than only when the whole batch is done. Closest TS shape: manually
  racing an array of promises with something like a `Promise.any`-in-a-loop
  pattern, or an async generator that yields as each settles (there's no
  single built-in with this exact incremental-drain shape).*
  Chapter: 3 (this is the second of the three named "distinct patterns" —
  `try_join_all` planner pattern vs. `FuturesUnordered` batch tool vs.
  `broadcast`+`select!` fan-in), 7 (bridges to `bus.rs`).

---

## Chapter 7 — Communication topology (continued): the global event bus

### `crates/jcode-base/src/bus.rs` (642 lines total)

Confirmed present, 642 lines. Test module starts at line 609
(`#[cfg(test)]`).

- **`BusEvent` enum** — lines 394-466. One sentence: a single large enum
  (~30 variants) covering every kind of cross-cutting, UI-facing event in
  the app — tool progress, batch progress, file touches, swarm output
  tails, background task completion, login/auth events, model catalog
  refresh, dictation, compaction, etc. Relevant swarm-adjacent variants:
  `BatchProgress(BatchProgress)` (399), `FileTouch(FileTouch)` (401, doc
  comment: "File was touched by an agent (for swarm conflict detection)"),
  `SwarmOutputTail(SwarmOutputTail)` (403, doc comment: "Streaming output
  tail from a swarm worker, for inline gallery viewports"),
  `SwarmAwaitCompleted(SwarmAwaitCompleted)` (409, doc comment: "A
  backgrounded `swarm await_members` watcher reached a terminal result" —
  this is exactly the type published from `comm_await.rs:196`, confirming
  the cross-file link noted in the Chapter 3 section above).
  Chapter: 7 (this enum is the entire vocabulary of the global bus — good
  "here's everything that flows through the one shared channel" exhibit).

- **`Bus` struct** — lines 468-474: just `sender: broadcast::Sender<BusEvent>`
  plus private debounce state for one specific event type. One sentence:
  the entire "bus" is a single tokio broadcast channel sender, confirming
  the plan's characterization "a `tokio::sync::broadcast` singleton."

- **`Bus::global()`** — lines 499-502.
  ```rust
  pub fn global() -> &'static Bus {
      static INSTANCE: OnceLock<Bus> = OnceLock::new();
      INSTANCE.get_or_init(Bus::new)
  }
  ```
  This is the exact singleton-accessor citation. **[RUST]** *plain English:
  `static INSTANCE: OnceLock<Bus> = OnceLock::new()` plus `get_or_init` is
  Rust's idiomatic lazy-singleton pattern — the `Bus` is created exactly
  once, the first time `global()` is called from anywhere in the process,
  and every subsequent call gets a reference to that same instance. This is
  the same idea as a JS module-level `let instance; export function
  getBus() { if (!instance) instance = new Bus(); return instance; }` — a
  process-wide singleton, not tied to any one request/session.*

- **`Bus::new`** — lines 511-519: `broadcast::channel(256)` — a
  **fixed-capacity 256-slot ring buffer**. Worth flagging for Chapter 7:
  this bounded capacity is exactly why `comm_await.rs`'s handling of
  `RecvError::Lagged` (see above) matters — a burst of >256 unconsumed
  events between two `.recv()` calls on any one subscriber will drop the
  oldest ones for that subscriber specifically (each subscriber gets its
  own read cursor into the same ring buffer; slow subscribers lag
  independently rather than blocking fast ones or the publisher).

- **`Bus::subscribe`** — lines 521-523, thin wrapper over
  `self.sender.subscribe()`.

- **`Bus::publish`** — lines 525-533: `let _ = self.sender.send(event);`
  (line 532) — note the `let _ =`: a `broadcast::Sender::send` returns
  `Err` only when there are zero receivers, and the bus deliberately
  ignores that (nobody being subscribed isn't an error for a fire-and-publish
  event bus). Also has a special case (lines 526-531) that additionally
  caches the latest `UpdateStatus` in a separate global `Mutex` so
  late-subscribing readers can poll "what's the current status" without
  having observed the original publish — a "broadcast channels have no
  replay/history" workaround worth flagging in Chapter 7 (contrast with
  `swarm.rs`'s `event_history: Arc<RwLock<VecDeque<SwarmEvent>>>`, which
  *does* keep replay history for the swarm-specific event stream).

- **`Bus::publish_models_updated`** — lines 542-606: a second, more
  elaborate debounce example (750ms window, `MODELS_UPDATED_DEBOUNCE` at
  line 476) — same coalescing idea as `swarm.rs`'s status-broadcast
  debounce, implemented independently here. Good "this pattern recurs" note
  for Chapter 5/7 without needing to read it line-by-line for the handbook.

- **`ToolStatus`/`ToolEvent`/`TodoEvent`/`FileTouch`/etc. structs** — lines
  16-393, the payload types carried by `BusEvent` variants. Not
  individually load-bearing for the handbook; skip unless a later chapter
  needs one specific payload shape (e.g. `SwarmAwaitCompleted` at lines
  281-294 — fields `session_id`, `completed`, `summary`, `notification`,
  `notify`, `wake`; `SwarmOutputTail` at lines 121-126 — `session_id`,
  `tail`).

---

## Chapter 7 — Communication topology (continued): `jcode-swarm-core`'s non-DAG contents

### `crates/jcode-swarm-core/src/lib.rs` (837 lines total)

Confirmed present, 837 lines. Test module starts at line 617
(`#[cfg(test)] mod tests`). **See the Chapter 6 "CRITICAL FINDING" section
above first** — this crate is NOT the DAG engine; what it actually
contains is: swarm-member/channel bookkeeping types, and LLM-facing
prompt-text generator functions. Both are genuinely useful for Chapter 7
(communication topology) and Chapter 2 (spawn — completion-report
contract), just not for Chapter 6.

- **Constants** — `MAX_SWARM_COMPLETION_REPORT_CHARS = 4000` (line 7),
  `SWARM_COMPLETION_REPORT_MARKER` (8), `SWARM_TLDR_REQUIRED_OVER_CHARS =
  240` (13), `MAX_SWARM_TLDR_CHARS = 200` (17), `MAX_SWARM_MEMBERS = 1000`
  (60, re-exported into `swarm.rs:32` as covered above),
  `MAX_SWARM_TASK_LABEL_CHARS = 48` (63), `SWARM_DEEP_NODE_MARKER` (378).

- **`validate_swarm_tldr`** — lines 25-56. One sentence: enforces that any
  message body over `SWARM_TLDR_REQUIRED_OVER_CHARS` (240 chars) carries a
  short (`MAX_SWARM_TLDR_CHARS` = 200 char) single-line `tldr`, returning a
  model-actionable error string otherwise. Chapter: 7 (message-size
  discipline for the communication layer — "status snapshot vs. summary vs.
  full context reads" from the plan's Chapter 7 outline).

- **`derive_swarm_task_label`** — lines 70-88. Derives a short UI label
  from a spawn prompt's first non-empty line. Chapter: 2 (labels shown in
  swarm member UI, referenced from `comm_session.rs:736`
  `set_member_task_label`).

- **`SwarmRole` enum** — lines 91-95 (`Agent`, `Coordinator`, `Other(String)`)
  with `as_str` (98-104), a hand-written `From<String>` (107-115), and a
  hand-written `Serialize` impl (117-124) plus `Deserialize` (126-133) —
  worth noting as a small Rust idiom: an open string-extensible enum
  (`Other(String)` catches unrecognized role strings from the wire rather
  than failing to deserialize).

- **`SwarmLifecycleStatus` enum** — lines 136-151 (`Spawned`, `Ready`,
  `Running`, `RunningStale`, `Completed`, `Done`, `Failed`, `Stopped`,
  `Crashed`, `Queued`, `Blocked`, `Pending`, `Todo`, `Other(String)`),
  same shape/pattern as `SwarmRole` (`as_str` 154-172, `From<String>`
  174-193, `Serialize` 195-202, `Deserialize` 204-211) — the member
  lifecycle states (beyond the plain `&str` status seen elsewhere in
  `swarm.rs`; note this enum and the bare `status: String` field used
  throughout `swarm.rs`/`comm_await.rs` appear to be two different
  representations of "member status" in this codebase — worth a Phase 2
  agent double-checking which one is authoritative before writing Chapter
  7, since this map didn't trace every call site).

- **`SwarmMemberRecord` struct** — lines 215-231 (`session_id`,
  `working_dir`, `swarm_id`, `swarm_enabled`, `status:
  SwarmLifecycleStatus`, `detail`, `task_label`, `friendly_name`,
  `report_back_to_session_id`, `latest_completion_report`, `role:
  SwarmRole`, `is_headless`). Doc comment (213): "Durable, persistable
  portion of a swarm member." One sentence: the on-disk-persisted subset of
  a live `SwarmMember`. **Follow-up now resolved**: the live struct is
  confirmed at `crates/jcode-app-core/src/server/state.rs:188` (`pub
  struct SwarmMember`, doc comment line 186 "Information about a session
  in a swarm"), and it is a genuinely different shape, not just a
  superset — notably `SwarmMember.status` (state.rs:204) is a **bare
  `String`** ("Lifecycle status (ready, running, completed, failed,
  stopped, etc.)"), NOT the `SwarmLifecycleStatus` enum used by the
  persisted `SwarmMemberRecord`. This confirms the discrepancy flagged
  above: the live in-memory struct uses untyped strings for status/role,
  while the persistence-layer record uses the typed enums — the enum is
  translated to/from the live string somewhere in the persistence code
  path (not traced further in this pass). `SwarmMember` also carries
  transport/live-only fields with no persisted counterpart: `event_tx`/
  `event_txs` (state.rs:194,196 — per-connection `mpsc` senders, doc
  comment 190-196 explains `event_tx` is kept for "backward-compatible
  single-sender call sites" while `event_txs` is the live multi-attachment
  map), `output_tail` (226-229, "not persisted" per its own doc comment),
  `todo_progress` (230-233, also "not persisted"). Note this struct is
  where `report_back_to_session_id` — the spawn-ancestry field central to
  Chapter 2 — is declared with its type (state.rs:214).

- **`ChannelIndex` struct** — lines 235-238 (`by_swarm_channel:
  HashMap<String, HashMap<String, HashSet<String>>>`, `by_session:
  HashMap<String, HashMap<String, HashSet<String>>>`). Doc comment (233):
  "Bidirectional index for swarm channel subscriptions." One sentence: a
  dual-indexed structure (swarm+channel -> members, AND session -> its
  swarm+channel subscriptions) so both "who's in this channel" and "what
  channels is this session in" are O(1)-ish lookups instead of a scan;
  methods `subscribe` (241-254), `unsubscribe` (256-286), `remove_session`
  (288-329, called on member departure to clean up both index directions),
  `members` (331-340). **This is the concrete data structure behind the
  plan's Chapter 7 "channels" bullet** — worth featuring directly, it's
  small, self-contained, and a clean two-map-mirror pattern to translate to
  a TS `Map<string, Set<string>>` pair.

- **`append_swarm_completion_report_instructions`** — lines 355-375. Body
  fully read above. One sentence: idempotently (checks
  `SWARM_COMPLETION_REPORT_MARKER` first, line 356) appends a
  `<system-reminder>` block to a spawned worker's initial prompt instructing
  it to call the swarm tool's `action="report"` before finishing. This is
  the function `comm_session.rs:620`
  (`initial_message.as_deref().map(append_swarm_completion_report_instructions)`)
  calls when preparing a spawn's startup message — direct citation link
  between Chapter 2 and Chapter 7's completion-report protocol.

- **`append_deep_node_instructions`** (390-433) and
  **`append_deep_gate_instructions`** (449-499) — covered in full under
  Chapter 6 above (these generate the deep-mode LLM directives, not DAG
  logic). Also relevant to Chapter 7 as "how does an agent learn the
  communication/action protocol" — the directive text itself names the
  swarm tool actions (`expand_node`, `complete_node`, `inject_gap`) an
  agent must call, which is the LLM-facing surface of the DAG engine
  covered in Chapter 6.

- **`format_structured_completion_report`** (501-522), **`normalize_completion_report`**
  (524-540, truncates over `MAX_SWARM_COMPLETION_REPORT_CHARS` = 4000 with
  a `"[Report truncated by jcode before delivery.]"` suffix, lines
  535-539), **`completion_status_intro`** (542-550, private, maps a raw
  status string to an intro sentence — note the `_ =>` catch-all at line
  548 means any status not in {ready, failed, stopped, crashed} still gets
  a generic "completed their work" message rather than erroring),
  **`completion_followup`** (552-576, private, maps `(status, has_report)`
  to a next-step suggestion string), **`completion_notification_message`**
  (578-585, the public composer — this is the function `comm_await.rs`
  imports and calls, confirmed via `swarm.rs`'s own import at lines 10-12
  `use jcode_swarm_core::{completion_notification_message,
  normalize_completion_report, truncate_detail};`).

- **`truncate_detail`** — lines 587-600. One sentence: whitespace-collapse
  then char-boundary-safe ellipsis-truncate — used throughout `swarm.rs`
  (e.g. line 371-377 in the dead-member salvage path) for short status/detail
  strings. Small but appears everywhere; worth one citation as "the shared
  string-shortening helper," not a chapter-defining function on its own.

- **`summarize_plan_items`** — lines 602-615, formats a `&[PlanItem]` (the
  `jcode-plan` type, imported at line 1) into a short joined-string
  preview, confirming again that `jcode-swarm-core` consumes `jcode-plan`'s
  types rather than defining its own competing plan-item type.

---

## Chapter 6 — The task DAG (deep mode)

### CRITICAL FINDING — re-scope this chapter around `crates/jcode-plan/src/dag/`, not `jcode-swarm-core`/`jcode-task-types`

The plan's exploration notes describe `jcode-swarm-core/src/lib.rs` and
`jcode-task-types/src/lib.rs` as owning "the recursive 'deep mode' task-DAG
types (`expand_node`/`complete_node`/`inject_gap`)." **Having opened all
three crates, this is materially wrong about where the DAG logic lives**,
and Phase 2/3 must not cite `jcode-swarm-core` or `jcode-task-types` as the
DAG engine:

1. **`jcode-task-types/src/lib.rs` (854 lines) is NOT about the swarm task
   DAG at all.** Verified by reading its full symbol list (`GoalScope`,
   `GoalStatus`, `GoalStep`, `GoalMilestone`, `GoalUpdate`, `Goal`,
   `TodoItem`, `TodoPlan`, `TodoPlanField`, `TodoPlanChange`, `TodoGoal`,
   `TodoGoalField`, `TodoGoalChange`, `PersistedCatchupState`,
   `CatchupBrief`, `IterationMaturity`) and grepping for "Node"/"TaskGraph"
   (zero hits). It's a **long-lived goal/todo/catch-up tracking** crate —
   confirmed by its only consumers: `crates/jcode-app-core/src/catchup.rs`,
   `crates/jcode-base/src/todo.rs`, `crates/jcode-base/src/goal.rs` (found
   via `grep -rl jcode_task_types crates/`). It shares no types with the
   swarm task-DAG. **Do not cite this file for Chapter 6.** If the
   handbook wants to mention it at all, it belongs (briefly, as an aside)
   near session/todo bookkeeping, not swarm architecture.

2. **`jcode-swarm-core/src/lib.rs` does not implement `expand_node`,
   `complete_node`, or `inject_gap`/`inject_from_gate` as Rust
   functions.** Grepping the file confirms `expand_node`/`complete_node`/
   `inject_gap` appear ONLY as **string literals inside LLM-facing prompt
   text** (inside `append_deep_node_instructions`, lines 390-433, and
   `append_deep_gate_instructions`, lines 449-499) — i.e. these are the
   *tool action names the directive tells the worker/gate agent to call*,
   not functions defined in this crate.

3. **The actual DAG engine — validated mutations, node/edge types, the
   scheduler — lives in `crates/jcode-plan/src/dag/`** (`mod.rs` 682
   lines, `ops.rs` 878 lines, `schedule.rs` 106 lines, `sim.rs` 155 lines,
   `tests.rs` 1392 lines — all confirmed present via `wc -l`). This is
   file #10 in the task brief, and it is correctly identified there as
   likely the real core — **confirmed true**. `pub fn expand_node` is at
   `crates/jcode-plan/src/dag/ops.rs:227`, `pub fn complete_node` at
   `ops.rs:377`, and the gap-injection function (named `inject_from_gate`,
   not `inject_gap` — the plan's naming was for the prompt-text action
   string, not the Rust fn) at `ops.rs:444`.

4. **The three crates' actual relationship**, confirmed via `Cargo.toml`
   dependency edges and `bridge.rs`'s own doc comment:
   - `jcode-plan` is the base: owns `TaskGraph`/`TaskNode`/`Mode`/
     `NodeKind`/etc. (the validated DAG engine, `src/dag/`) **and** the
     older/live `VersionedPlan`/`PlanItem` model (`src/lib.rs:150`,
     `src/lib.rs:20` — this is the type `swarm.rs` and `comm_await.rs`
     operate on directly, e.g. `swarm_plans: Arc<RwLock<HashMap<String,
     VersionedPlan>>>`).
   - `jcode-swarm-core` depends on `jcode-plan` (`Cargo.toml:8`,
     `jcode-plan = { path = "../jcode-plan" }`; confirmed by `use
     jcode_plan::PlanItem;` at `jcode-swarm-core/src/lib.rs:1`) and adds
     swarm-member bookkeeping types (`SwarmMemberRecord`, `SwarmRole`,
     `SwarmLifecycleStatus`, `ChannelIndex`) plus **prompt-text generator
     functions** that narrate the DAG protocol to LLM workers
     (`append_deep_node_instructions`, `append_deep_gate_instructions`,
     `append_swarm_completion_report_instructions`) and small
     presentation helpers (`truncate_detail`, `derive_swarm_task_label`,
     completion-report formatting). It does not own DAG mutation logic.
   - `jcode-app-core` (where `swarm.rs`/`comm_session.rs`/`comm_await.rs`
     live) depends on **both** `jcode-plan` and `jcode-swarm-core`
     (confirmed in `Cargo.toml:94-95`) — i.e. it's the consumer at the top,
     wiring the validated engine and the prompt-text helpers into the live
     server.
   - `jcode-task-types` is a sibling, unrelated dependency, also consumed
     by `jcode-app-core` (`Cargo.toml:99`) but for goals/todos, not swarm.
   The clearest single citation for this whole relationship is
   **`crates/jcode-plan/src/bridge.rs:1-9`**, a doc comment worth quoting
   near-verbatim in Chapter 6:
   > "Bridge between the validated `crate::dag` engine and the live
   > `VersionedPlan` storage used by the swarm runtime. The `dag` engine is
   > the brain: it owns validation (acyclicity, ownership, gate insertion,
   > artifact checks) and the reference simulator. `VersionedPlan` is the
   > live, persisted, broadcast storage. Rather than run two parallel
   > runtimes, server handlers lift the current plan into a `TaskGraph`,
   > apply an engine op, then lower the result back. This keeps a single
   > source of truth and reuses the existing persistence/broadcast/scheduler
   > machinery."
   **[RUST]** *plain English: "lift into a TaskGraph, apply an op, lower
   back" describes a **pure-functional core wrapped by a stateful shell**
   pattern — the validated engine (`dag/`) has no side effects and no
   knowledge of the server; `bridge.rs` is the adapter that converts the
   server's live, persisted `VersionedPlan` struct into the engine's
   `TaskGraph`, calls a pure engine function, and converts the result back.
   This is a very portable teaching pattern for TS: keep your core logic as
   pure functions over plain data, and isolate all I/O/mutation in a thin
   adapter layer around it.*

### `crates/jcode-plan/src/dag/mod.rs` (682 lines total — full file read)

Confirmed present, 682 lines. Crate/module doc comment (lines 1-10) is
itself a great Chapter 6 opening quote:
> "Task-DAG engine model. This is the DAG-first reframe of swarm described
> in `docs/SWARM_TASK_GRAPH.md`. The graph is the primary object: nodes are
> tasks, edges are dependencies, and agents are fungible workers that
> execute, decompose (composite nodes), and verify (gate nodes) those
> tasks. The model here is deliberately decoupled from the server/runtime
> wiring so it can be exercised end-to-end by the deterministic simulator
> in [`crate::dag::sim`] before being attached to live swarm sessions."

- **`Mode` enum** — lines 36-49 (`Deep`, `Light`), with
  `requires_gates()` (46-48). Doc comment (33-35): "Engine mode. One
  engine, two presets... The data model, scheduler, and dataflow are
  identical; the mode only controls whether the rigor machinery (mandatory
  gates + strict artifact validation) is engaged." **This is the `Mode`
  enum named in the task brief** — confirmed to exist exactly as described
  (Deep/Light presets). Chapter: 6 (primary), 2 (mode-gated spawning — note
  again this is a *different* "mode" axis than `SwarmSpawnMode` in
  `comm_session.rs`; the handbook should clearly separate "how is the
  worker window rendered" (`SwarmSpawnMode`) from "how rigorous is the
  task-graph machinery" (`dag::Mode`)).

- **`NodeOrigin` enum** — lines 58-67 (`Seed`, `Expand`, `Gap`, `Gate`).
  Doc comment (51-55) explains the growth-accounting rationale: "Deep
  mode's growth pressure is measured against this: `Seed` nodes are the
  first agent's draft, everything else is growth the machinery generated...
  Status surfaces report seeded-vs-grown so a plan that never outgrew its
  seed is visibly under-explored."

- **`NodeKind` enum** — lines 72-85 (`Explore`, `Implement`, `Verify`,
  `Fix`, `Synthesize`, `Critique`), with **`is_gate_kind`** (89-91) and
  **`gate_kind`** (96-101, task brief calls this `gate_kind()` —
  confirmed exact name and location). One sentence: `gate_kind()` maps a
  node's kind to the kind of auto-inserted gate that must guard it —
  exploration-style work (`Explore`/`Synthesize`/`Critique` fall through
  the `_` arm) gets a `Critique` gate, code-style work (`Implement`/`Fix`)
  gets a `Verify` gate (lines 97-100).

- **`NodeStatus` enum** — lines 107-116 (`Queued`, `Running`, `Done`,
  `Failed`). Doc comment (104-105) — good nuance-flag: "'Blocked' is
  intentionally not stored: it is computed from dependency state by the
  scheduler, so there is a single source of truth." Chapter: 6 (single-
  source-of-truth pattern, good contrast to naive status-flag designs).

- **`ConfidenceLevel` enum** — lines 129-133 (`Low`, `Medium`, `High`),
  with **`parse`** (141-204, a genuinely elaborate lenient free-text parser
  handling word rungs, negations, percentages, fractions, and 0-1/0-10/
  0-100 scores) and **`as_str`** (206-212). Doc comment (118-127) is a
  strong Chapter 6 "why" quote:
  > "Confidence is the breadth signal of the task graph: a node completed
  > at `ConfidenceLevel::Low` is an admission that its scope was not
  > adequately covered, so the machinery treats it like
  > `what_i_did_not_check` — gates are pointed at low-confidence siblings
  > and (in deep mode) cannot pass while such a sibling is unaddressed."
  **This is exactly the `ConfidenceLevel` parsing the task brief calls
  out** — confirmed present with this signature and this design
  rationale.

- **`HandoffArtifact` struct** — lines 260-284 (`findings`, `evidence`,
  `edge_cases_considered`, `validation`, `open_questions`, `confidence`,
  `what_i_did_not_check`), with **`brief`** (288-293), **`confidence_level`**
  (297-299), and **`render_section`** (310-344, the "single source of
  truth for how an artifact is surfaced on a dependency edge," doc comment
  301-309). One sentence: the typed handoff payload a node attaches on
  completion — the "dataflow" that moves forward along `depends_on` edges
  to dependent/downstream nodes and gates. Chapter: 6 — this is the
  concrete artifact contract worth a full snippet.

- **`TaskNode` struct** — lines 349-388 (`id`, `content`, `kind`, `status`,
  `owner`, `parent`, `depends_on`, `expanded`, `is_gate`, `planner`,
  `priority`, `output`, `origin`), with **`is_composite`/`is_done`/
  `is_terminal`** (391-401).

- **`NodeSpec` struct** — lines 407-416 plus builder methods
  `new`/`depends_on`/`priority` (419-437) — the input shape callers use to
  add nodes (id may be omitted for auto-assignment).

- **`DagError` enum** — lines 442-471, 11 variants covering every way a
  mutation can be rejected (`UnknownNode`, `DuplicateNode`,
  `UnknownDependency`, `WouldCreateCycle`, `NotOwner`, `InvalidState`,
  `ThinArtifact`, `UnaddressedLowConfidence`, `UncoveredSiblings`,
  `StaleGateScope`, `GateMisuse`), with a `Display` impl (473-531) whose
  error strings double as
  LLM-facing correction messages (e.g. `UnaddressedLowConfidence` at
  500-509 tells the gate agent exactly what to do next: "inject_gap with
  follow-up nodes... or name each id in your findings"). Good Chapter 6
  exhibit for "validation errors as structured, actionable feedback to the
  calling agent," a nice structural echo of the handbook's own
  anti-hallucination citation-gate theme (as the plan's outline for
  Chapter 6 already anticipates).

- **`TaskGraph` struct** — lines 539-542 (`mode: Mode`, private `nodes:
  Vec<TaskNode>`), with accessor/query methods: `new` (545-550), `nodes`
  (552-554), `len`/`is_empty` (556-562), `get`/`get_mut` (564-570),
  `contains` (572-574), `push`/`push_node` (576-585), `children_of`
  (588-593), `gate_of` (596-600), **`low_confidence_done_ids`** (607-619,
  doc comment 602-606 explains this is the "shaky coverage" set gates
  prioritize), **`all_terminal`** (622-624), and **`cycle_nodes`**
  (628-681, an explicit Kahn's-algorithm topological-sort cycle detector
  with an inline comment on why it dedupes `depends_on` edges before
  counting in-degree, lines 638-650).
  **[RUST]** *plain English: `nodes: Vec<TaskNode>` is a private field
  (no `pub`) — external code can only read/mutate the graph through the
  methods listed above. This is Rust's version of encapsulation
  (equivalent to a JS/TS class with a private field and only getter/
  mutator methods exposed) — it's how the crate enforces that all
  mutations go through the validated `ops::` functions rather than letting
  callers splice the node list directly.*

### `crates/jcode-plan/src/dag/ops.rs` (878 lines total — grepped for signatures, not fully read line-by-line; cite cautiously, re-verify exact bodies before quoting code)

Confirmed present, 878 lines. Module doc-comment-level design rationale
(each function has a substantial doc comment; only summarized here, not
the full bodies):

- **`seed`** — `pub fn seed(graph: &mut TaskGraph, specs: Vec<NodeSpec>) ->
  Result<(), DagError>` at line 19 (doc comment 14-18: seeds the initial
  DAG from the first agent's draft batch; replaying an identical seed is a
  no-op "which makes transport/tool retries safe").
- **`ensure_root_gate`** — private fn at line 143 (doc comment 135-142):
  inserts/refreshes the deep-mode root gate auditing the current root node
  set; re-queues it if new work arrives after it went terminal ("a
  re-seeded plan can never stay 'finished' unaudited").
- **`ExpandOutcome` struct** — line 213 (doc comment 211: "The result of
  expanding a node into children").
- **`expand_node`** — `pub fn expand_node(...)` at **line 227** (task
  brief named this function; confirmed present at this exact line). Doc
  comment (220-226): "Decompose a node the actor owns into a child sub-DAG
  (the composite path). The node flips to composite and becomes a
  join/synthesis point that depends on its children. In deep mode a
  critique/verify gate is auto-inserted between the children and the
  synthesis, so the composite cannot close without surviving it."
- **`complete_node`** — `pub fn complete_node(...)` at **line 377**
  (confirmed). Doc comment (369-376): artifact is validated for
  "thinness" in deep mode; a gate additionally may not pass while a
  sibling completed with low confidence unless explicitly addressed;
  escape hatch is `inject_from_gate`.
- **`fail_node`** — `pub fn fail_node(graph: &mut TaskGraph, node_id: &str,
  actor: &str) -> Result<(), DagError>` at line 415.
- **`inject_from_gate`** — `pub fn inject_from_gate(...)` at **line 444**
  (this is the Rust function behind the prompt-text `inject_gap` action
  name). Doc comment (438-443): "Inject new gap/fix nodes from a gate that
  found a problem (the adversarial path). The gate does not decompose
  itself; instead it adds new sibling nodes under the same composite
  parent and re-queues itself to depend on them."
- **`requeue_failed`** — line 547 (doc comment 542-546): the retry path;
  clears ownership so the retry can go to any worker.
- **`GATE_COVERAGE_ENUMERATION_CAP`** — a `pub` const re-exported from
  `mod.rs:22`; per a comment near line 749-750 in `ops.rs`, above this many
  audited nodes a passing gate artifact no longer has to enumerate every
  id.
- Helper internals: `unique_gate_id` (569), `mentions_node_id` (593, a
  careful word-boundary matcher — doc comment 584-592 explains why naive
  substring `contains` would be a real bug: "a short child id like 'a' or
  'fix' would match nearly any English sentence and let a gate rubber-stamp
  an unaddressed low-confidence sibling"), `validated_spec_id` (633),
  `spec_to_node` (646), `gate_content`/`root_gate_content` (673, 688),
  `validate_artifact` (703).

### `crates/jcode-plan/src/dag/schedule.rs` (106 lines total — full file read)

Confirmed present, 106 lines. Module doc comment (1-6): "Scheduler:
ready-set computation, dispatch, and dataflow hydration. The scheduler
walks the DAG. A node becomes runnable when all its dependencies are
`Done`. On dispatch it is assigned to a worker (ownership) and its input is
hydrated from the merged artifacts of its upstream dependencies, which is
the forward dataflow along edges."

- **`LIGHT_MODE_SUGGESTED_WORKERS`** — line 12, `pub const ... : usize =
  16`. Doc comment (10-11): "Suggested default worker ceiling for light
  mode... Deep mode is bounded by the swarm-level `MAX_SWARM_MEMBERS` cap
  instead." Good cross-reference to `jcode-swarm-core::MAX_SWARM_MEMBERS`
  (= 1000, `jcode-swarm-core/src/lib.rs:60`) re-exported through
  `swarm.rs:32`.
- **`is_terminal`** — lines 15-17, thin wrapper over `TaskNode::is_terminal`.
- **`ready_nodes`** — lines 21-29. One sentence: the set of `Queued` nodes
  whose every dependency is `Done`, sorted by `(priority, id)` for
  deterministic ordering — the DAG-engine equivalent of `swarm.rs`'s
  planner fan-out list, but computed from graph structure rather than an
  LLM's flat JSON plan.
- **`deps_satisfied`** — lines 31-40 (private), the dependency-check
  predicate `ready_nodes` and `dispatch` both use.
- **`dispatch`** — lines 44-59. One sentence: assigns a ready node to a
  named worker and flips it `Queued -> Running`; returns `false` (not an
  error type) if the node wasn't actually dispatchable — a check-then-act
  pattern done carefully to avoid a race between the dispatchability check
  and the mutation (comment at line 52: "`dispatchable` proved the node
  exists under this same borrow of `graph`").
- **`assemble_input`** — lines 64-92. One sentence: builds a worker's full
  input by concatenating the node's own prompt with a rendered section
  (via `HandoffArtifact::render_section`) for every completed upstream
  dependency — this *is* the forward-dataflow mechanism the DAG uses
  instead of message-passing between agents.
- **`kind_label`** — lines 96-106, private lowercase-string mapper for
  `NodeKind`, kept in sync with `bridge.rs`'s `kind_str` (per its own
  comment, line 95).

### `crates/jcode-plan/src/dag/sim.rs` (155 lines) and `tests.rs` (1392 lines)

Confirmed present via `wc -l` (155, 1392). Not read in detail — `sim.rs` is
referenced by `mod.rs`'s crate doc comment as "the deterministic simulator"
used to exercise the engine end-to-end before wiring to live sessions;
`tests.rs` is the engine's unit-test suite (1392 lines — larger than the
engine itself, suggesting fairly thorough coverage of the `DagError`
variants and gate-rejection paths). Worth a Phase 2 agent skimming `sim.rs`
directly if Chapter 6 wants a "how would I run this DAG without the whole
server" example; not verified further here due to scope.

### `crates/jcode-plan/src/bridge.rs` (503 lines) — the live/engine adapter

Confirmed present, 503 lines. Already quoted above (lines 1-9) for the
jcode-plan/jcode-swarm-core/jcode-task-types relationship. Additional
verified symbols: `parse_mode`/`mode_str` (15, 22), `parse_kind`/`kind_str`
(30, 41), `parse_origin`/`origin_str` (54, 64), `to_task_graph` (94, "Lift
a `VersionedPlan` into a validated `TaskGraph`"), `apply_task_graph` (124,
"Lower a `TaskGraph` back into the plan's items + node_meta"),
`upstream_context` (188, doc comment 180-187: "the live counterpart of
`dag::assemble_input`, but it reads artifacts from the plan's `node_meta`
side-map instead of a `TaskGraph`, so it can run directly on the assignment
path without lifting the whole graph" — i.e. a deliberate shortcut/
duplication to avoid the lift cost on the hot assignment path).

### `crates/jcode-plan/src/lib.rs` (1201 lines) — the live plan model `dag/` bridges into

Confirmed present, 1201 lines. Not read in full; key symbols confirmed via
grep: **`VersionedPlan` struct — line 150** (the type `swarm.rs` and
`comm_await.rs` hold in `Arc<RwLock<HashMap<String, VersionedPlan>>>`),
**`PlanItem` struct — line 20** (the flat, string-status task-list item
type `swarm.rs`'s planner/fan-out code and `newly_ready_item_ids` operate
on — imported into `swarm.rs` at line 5), `NodeMeta` (117),
`SwarmTaskProgress` (37), `TaskControlAction` enum (301),
`is_terminal_status`/`is_completed_status`/etc. (275-296, plain-string
status predicates — note `PlanItem.status` is a bare `String`, not the
`dag::NodeStatus` enum; the bridge's `status_from_plan`/`status_to_plan`
in `bridge.rs:74-93` are exactly the translation layer between these two
different status representations).

---

## Chapter 8 — Design-pattern glossary & further reading: the two design docs

### `docs/SWARM_ARCHITECTURE.md` (318 lines total — full file read)

Confirmed present, 318 lines. Header (line 3-5) is itself the single most
important framing fact for the handbook:
> "Status: Largely implemented (see `SWARM_TASK_GRAPH.md` for the DAG-first
> model that supersedes the agent-first framing here; its staged comm
> migration is in progress)"
**I.e. this doc describes the OLDER, agent-first model, partially
superseded by `SWARM_TASK_GRAPH.md`'s DAG-first model.** The handbook
should present `SWARM_ARCHITECTURE.md` content as historical/foundational
context (roles, lifecycle, communication vocabulary still in active code)
but flag the plan-mutation/coordinator-centric framing as the pre-DAG
picture where `SWARM_TASK_GRAPH.md` disagrees.

Section-to-chapter map (all section headers verified present at these
approximate line numbers by reading the full file):
- **Goals** (10-18): Chapter 1 (framing) — "Parallel work across many agents
  without locks," "optimistic by default" language feeds directly into
  Chapter 5's "shared state without locks-as-a-crutch" nuance the plan
  flags (contrast against the actual `RwLock` usage in `runtime.rs`/
  `swarm.rs`).
- **Roles > Mode-gated spawning** (22-58): Chapter 2. This section's prose
  is the direct source for the plan's Chapter 2 description and matches
  what was verified in code: "The spawn/parent edge is encoded by
  `report_back_to_session_id`" (line 34) = `swarm.rs::swarm_ancestors`
  (41-60); "Normal ad hoc swarms and light-swarm mode are one-level
  fan-out: only the root session may spawn agents" (24-25) = the
  `MAX_SWARM_MEMBERS` doc comment in `swarm.rs:26-31`; "An agent may stop
  any agent in its own subtree" (36-37) = `swarm_is_self_or_ancestor`
  (`swarm.rs:76-85`). Also documents **reparenting on mid-tree departure**
  (40-45, "its direct children are reparented... attach to their live
  grandparent, falling back to the current coordinator") — this behavior
  was NOT independently verified in the source files opened for this map;
  flag for Phase 2 to locate and cite the actual reparenting code if the
  handbook wants to claim it (likely in `remove_session_from_swarm`,
  `swarm.rs:995-1219`, not read in full above).
- **Roles > Coordinator / Worktree Manager / Agents** (60-87): Chapter 1
  (vocabulary), Chapter 7.
- **Agent Lifecycle States** (89-98) and **Agent Lifecycle Notifications**
  (100-106): Chapter 7 — cross-check against `SwarmLifecycleStatus`
  (`jcode-swarm-core/src/lib.rs:136-151`): the doc lists `spawned, ready,
  running, blocked, completed, failed, stopped, crashed` (8 states); the
  enum has 13 named variants plus `Other(String)` (`Spawned, Ready,
  Running, RunningStale, Completed, Done, Failed, Stopped, Crashed,
  Queued, Blocked, Pending, Todo`) — **the doc and the enum do not match
  1:1** (enum has `RunningStale`/`Done`/`Queued`/`Pending`/`Todo` not in
  the doc's list). Flag this as a real discrepancy for Phase 2/4: cite the
  enum as authoritative for "what statuses exist in code," the doc for
  "what the design intends," and don't claim they're the same list.
- **Completion Report Policy** (108-124): Chapter 2/7 — matches
  `jcode-swarm-core::append_swarm_completion_report_instructions`
  (355-375) and `completion_notification_message` (578-585) almost
  exactly in spirit ("should include outcome/status, changes or findings,
  validation performed, and blockers or follow-ups" (114) mirrors the
  prompt text at `swarm-core` lines 367-368).
- **User Interaction** (126-130): Chapter 1.
- **Plan Distribution and Updates** (132-151, incl. mermaid flowchart):
  Chapter 7 — matches `swarm.rs::broadcast_swarm_plan`/
  `send_swarm_plan_to_session` (818-970) closely: "Plan updates are
  propagated to plan participants, not every agent in the swarm" (138) =
  the participants-vs-fallback-to-all-members logic at `swarm.rs:872-882`.
- **Worktree Usage** (153-192, two mermaid diagrams): NOT covered by any
  file opened in this exploration pass — no worktree-manager source file
  was in the task brief's file list. Flag: if the handbook's Chapter 1
  diagram wants to mention worktrees, it should cite this doc only (prose
  claim, not verified against source in this map) or a Phase 2 agent
  should locate the worktree-manager implementation first.
- **Communication** (194-254, incl. mermaid): Chapter 7 — direct source
  for "DMs, subtree broadcast, channels" from the plan's Chapter 7 outline,
  and the "status snapshot vs. summary vs. full context reads" distinction
  (221-229) named verbatim in the plan. Note: `docs/SWARM_TASK_GRAPH.md`
  §8a (below) documents channels and shared-context as **being
  deprecated/cut** — this doc's Communication section describes the
  pre-deprecation richer picture; the handbook should present the
  DAG-first doc's leaner two-tier model (structural dataflow + DM/subtree
  broadcast) as current direction, with this doc's channel/shared-context
  material flagged as "being phased out," not as equally-current.
- **UI (TUI)** (256-298, two mermaid diagrams): Chapter 1 (maybe one
  diagram source), not deeply relevant to the concurrency/pattern chapters.
- **File Touch and Intent** (300-305): Chapter 7 (file-touch conflict
  detection — matches `BusEvent::FileTouch`, `jcode-base/src/bus.rs:401`).
- **Conflict Handling (No Locks)** (307-311): **This is the exact
  "optimistic, no locks" language the plan's Chapter 5 wants to nuance**:
  "The system is optimistic by default (no locks)... Coordination happens
  via DM or channel, not through the coordinator." Chapter 5 must frame
  this precisely: it is true at the **coordination-protocol level** (no
  agent blocks waiting for a lock to edit a file or claim work — conflicts
  are resolved by convention/communication, not mutual exclusion), while
  the **implementation** uses `RwLock`/`Mutex` extensively for its own
  in-memory data structures (member registries, plan storage, event
  history — see every `Arc<RwLock<HashMap<...>>>` field in
  `runtime.rs:90-120` and the function signatures throughout `swarm.rs`).
  These are two different senses of "lock" and the handbook should say so
  explicitly rather than let the doc's line 309 read as a contradiction of
  the code.
- **Summary** (313-318): Chapter 1/8 closing framing.

### `docs/SWARM_TASK_GRAPH.md` (605 lines total — full file read)

Confirmed present, 605 lines. Header (lines 3-6) is the second half of the
framing pair:
> "Status: Being implemented (supersedes the agent-first framing in
> `SWARM_ARCHITECTURE.md`). The DAG engine, deep/light modes, gates,
> growth mechanics, and comm migration steps 1-2 (artifact dataflow,
> subtree-scoped broadcast) are live; channel/shared-context deprecation
> (steps 3-4) is pending."
This status line is itself a precise, quotable statement of exactly how
"live vs. aspirational" each part of this doc is — worth citing directly
rather than paraphrasing, since it tells the handbook exactly which
sections describe shipped behavior (§1-7, migration steps 1-2) vs. planned
future work (migration steps 3-4, and arguably §8's "proposed tool
surface" — the task brief's file list does confirm `expand_node`/
`complete_node` exist as real ops in `jcode-plan/src/dag/ops.rs`, so §8's
core claims are implemented, but exact current tool-call names/shapes
should be re-verified against the live `swarm` tool schema before Chapter
6 asserts them, since this map did not open the tool-schema file).

Section-to-chapter map:
- **§1 Motivation and core reframe** (16-50, mermaid diagram): Chapter 1
  (framing — "agent-first" vs "DAG-first" is a good one-paragraph
  contrast for the handbook's opening) and Chapter 6.
- **§1a Two modes: deep vs light** (54-98, comparison table): **Chapter 6,
  primary source for `dag::Mode`.** Directly matches the verified
  `Mode` enum (`jcode-plan/src/dag/mod.rs:37-49`) and its doc comment
  almost word for word ("one engine, two presets," "the data model,
  scheduler, and dataflow are identical; the mode only controls whether
  the rigor machinery... is engaged" — doc line 56-60 vs. code comment
  `dag/mod.rs:33-35`, near-identical phrasing, confirming the doc and code
  are in sync here). The comparison table (84-93) is handbook-ready as-is
  for a "Deep vs Light at a glance" sidebar, though the member-cap numbers
  should be cross-checked against `LIGHT_MODE_SUGGESTED_WORKERS = 16`
  (`schedule.rs:12`) and `MAX_SWARM_MEMBERS = 1000`
  (`jcode-swarm-core/src/lib.rs:60`) — both confirmed to match the doc's
  "small (e.g. 4-16 workers)" and "up to 1000 agents."
- **§2 Ownership tree over a dependency graph** (102-121): Chapter 5/6 —
  "The unit of mutation is *expanding a node you own*, never editing
  arbitrary nodes" (110) is the design rationale behind `ops::expand_node`
  and `ops::complete_node`'s ownership checks (`DagError::NotOwner`,
  `dag/mod.rs:452`).
- **§3 Node kinds: atomic vs composite** (124-157, mermaid diagram):
  Chapter 6 — matches `TaskNode.expanded`/`is_composite()`
  (`dag/mod.rs:369, 391-393`) and the map-reduce synthesis description
  matches `ops::expand_node`'s doc comment.
- **§4 Node kinds by terminal action** (160-191, table + mermaid): Chapter
  6 — directly matches `NodeKind` (`dag/mod.rs:72-85`), though note the
  doc's table only lists `explore/implement/verify/fix` (4 kinds); the
  code enum has 6 variants (`Synthesize`, `Critique` are the other two,
  `dag/mod.rs:82,84`) — minor doc/code drift, not a contradiction (the doc
  predates or just omits the gate-kind variants from this particular
  table), worth a one-line footnote rather than treating as an error.
- **§5 Dataflow: how a finished dependency passes off information**
  (195-216): **Chapter 6/7, primary source for the "dependency edge IS the
  data channel" idea.** Directly matches `schedule::assemble_input`
  (`dag/schedule.rs:64-92`) and `HandoffArtifact::render_section`
  (`dag/mod.rs:310-344`) — "by-reference, not by-value" (207) matches the
  artifact fields being file:line/commit refs (`evidence: Vec<String>`,
  `HandoffArtifact` struct).
- **§6 Completion and coverage** (219-304, incl. two mermaid diagrams):
  **Chapter 6, primary.** §6.1-6.3 are the prose design rationale;
  **§6.4 "Implemented enforcement (2026-07: growth mechanics)"
  (275-304) is the single best passage in either doc for Chapter 6** — it
  explicitly names the real engine mechanics and error variants: "Root
  gate," `plan::gate`, `inject_gap`, "Enumerated gate coverage... up to an
  enumeration cap of 20" (matches `GATE_COVERAGE_ENUMERATION_CAP` in
  `ops.rs`, re-exported `dag/mod.rs:22`), `UncoveredSiblings`,
  `StaleGateScope` (both confirmed exact `DagError` variant names,
  `dag/mod.rs:460-468`), `no_artifact_requeues`, `expand_node`/
  `complete_node`, and growth-accounting via node `origin` (matches
  `NodeOrigin` enum, `dag/mod.rs:58-67`). This paragraph should be treated
  as close to ground truth and is safe to quote/paraphrase heavily for
  Chapter 6, since its specific claims were independently corroborated
  against the actual enum/const names in `dag/mod.rs` and `ops.rs`.
- **§7 Bias budget: what is fixed vs emergent** (307-354, mermaid
  diagram): Chapter 6 — good "design tradeoff" material, no direct code
  citation needed (it's a philosophy/rationale section); safe to paraphrase
  as design intent.
- **§8 Interface: enforced graph API, not an agent script** (358-403,
  mermaid diagram): Chapter 6 — "Proposed tool surface" (389-398) lists
  `swarm task_graph`, `swarm expand_node`, `swarm complete_node`, `swarm
  run` as tool-call shapes; **these are proposed/evolution language in the
  doc ("Proposed tool surface (evolution of `swarm`)", line 389) — Phase 2
  should verify the CURRENT actual tool action names by reading the live
  `swarm` tool's parameter schema (not opened in this pass) before
  presenting these as the literal current API**, since the doc itself
  hedges with "proposed."
- **§8a Communication rework: dataflow first, chat second** (407-475):
  **Chapter 7, primary source for the DM/subtree-broadcast/channel-
  deprecation narrative.** The "Staged migration" list (464-474) is
  precisely dated: "1. Done... 2. Done... 3. Migrate existing flows off
  channels/shared-context... 4. Deprecate, then remove..." — steps 1-2 are
  marked done, matching the header's status line; steps 3-4 are still
  pending as of this doc. This section directly supports (and slightly
  updates/corrects) `SWARM_ARCHITECTURE.md`'s richer Communication section
  above — the handbook's Chapter 7 should lead with this doc's leaner
  model and treat `SWARM_ARCHITECTURE.md`'s channel/shared-context
  material as "legacy surface still present but being phased out," per
  this doc's own framing.
- **§9 Worked example: graph evolution over time** (478-549, five mermaid
  diagrams across T0-T7): **This is the exact worked example the plan's
  Chapter 6 outline calls out** ("the worked example from
  `SWARM_TASK_GRAPH.md` §9"). Confirmed present, T0 through T7, following
  a concrete "explore multimonitor support in scrollwm" scenario through
  seed -> expand -> dispatch -> recursive self-decomposition -> synthesis
  -> critique-finds-gap -> re-critique -> final synthesize. Directly
  usable as Chapter 6's central narrative example; every step maps to a
  real engine operation covered above (`seed`, `expand_node`,
  `complete_node`, `inject_from_gate`).
- **§10 Data model changes (against `jcode-plan`)** (553-589): Chapter 6 —
  "Reuse `VersionedPlan`/`PlanItem`... Add: `PlanItem`: `owner_session`,
  `kind: atomic | composite`..." (555-561) — **cross-check note**: this
  describes extending `PlanItem` (the `jcode-plan/src/lib.rs:20` flat/live
  type) with DAG-shaped fields, which is a different design than what was
  actually found in `jcode-plan/src/dag/mod.rs` (a wholly separate
  `TaskGraph`/`TaskNode` type, bridged via `bridge.rs`, rather than
  `PlanItem` itself growing DAG fields). **This section may describe an
  earlier design iteration that was superseded by the actual
  `dag`-module-as-separate-engine-with-a-bridge implementation** — Phase 2
  should not claim `PlanItem` literally has an `owner_session`/`kind`
  field without independently grep-checking `jcode-plan/src/lib.rs`'s
  `PlanItem` struct (this map only confirmed `PlanItem` exists at line 20,
  not its full field list). Also contains the **"Runaway prevention: a
  single total-member cap"** subsection (572-580) with a specific,
  checkable claim: "implemented in `ensure_spawn_coordinator_swarm`
  (`server/comm_session.rs`)" — this matches a real function confirmed to
  exist in this map's `comm_session.rs` symbol list (line 1234), good
  citation for Chapter 2/6.
- **§11 Suggested build order** (593-605): Chapter 8 (glossary/further
  reading) — a roadmap list, useful only as historical context ("this is
  the order it was built in"), not as a claim about current state; items
  1-4 appear to be done per the rest of the doc, item 5's "Reframe the
  tool surface" and item 6's "Update `SWARM_ARCHITECTURE.md`" are
  self-referential to-dos whose completion status this map cannot verify
  without re-reading the live tool schema and re-checking
  `SWARM_ARCHITECTURE.md`'s own header (which, per above, still frames
  itself as superseded-but-present, suggesting item 6 is only partially
  done).

---

## Chapter 4 — Structured concurrency & cancellation

### `crates/jcode-app-core/src/server/runtime.rs` (484 lines total)

Confirmed present, 484 lines.

- **`RuntimeTaskScope` struct** — lines 33-37.
  ```rust
  #[derive(Default)]
  struct RuntimeTaskScope {
      cancellation: CancellationToken,
      tasks: Mutex<JoinSet<()>>,
  }
  ```
  One sentence: a container that owns every spawned connection/background
  task for a server runtime, paired with one `CancellationToken` that fans
  out shutdown to all of them.
  **[RUST]** *plain English: `JoinSet<()>` is a collection that lets you
  spawn many concurrent tasks and later wait on/collect all of them as a
  group (a bit like `Promise.all`, but tasks can be added dynamically and
  polled for completion one at a time). `CancellationToken` is a shareable
  "please stop" flag — cloning it (or making a `child_token()`) gives you a
  linked token that also fires when the parent fires, which is how one
  shutdown signal propagates to many tasks at once (conceptually similar to
  `AbortController`/`AbortSignal`).*
  Doc comment (lines 27-32), worth quoting for design rationale:
  > "Owns every connection task spawned by a server runtime. Dropping a
  > `JoinHandle` detaches its task, so accepting a connection must not
  > discard the handle. This scope gives the accept loops and their children
  > one cancellation boundary and lets server shutdown wait until all
  > children have observed cancellation and released their resources."
  Chapter: 4 (structured concurrency & cancellation). Also useful for
  Chapter 1's architecture diagram (it's the shutdown boundary for the whole
  server).

- **`RuntimeTaskScope::spawn`** — lines 40-59 (`async fn spawn<F, Fut>`).
  One sentence: registers a new task under the scope's cancellation token,
  first opportunistically reaping finished tasks, and refuses to spawn once
  the scope is already cancelled (returns `bool` = whether it was accepted).
  **[RUST]** *plain English: `F: FnOnce(CancellationToken) -> Fut` is a
  generic constraint saying "give me a function that takes a cancellation
  token and returns a future" — the scope hands each task its own linked
  child token so the task can watch for cancellation. `Fut: Future<Output =
  ()> + Send + 'static` means that future must be safe to move across
  threads (`Send`) and not borrow any short-lived data (`'static`) — both
  required because tokio's scheduler may run it on any worker thread at any
  later time.*
  Chapter: 4.

- **`RuntimeTaskScope::shutdown`** — lines 61-74.
  One sentence: flips the cancellation token, then drains the `JoinSet` out
  from under the mutex (via `std::mem::take`) before awaiting every child so
  a task that's mid-registration doesn't deadlock against the shutdown
  routine.
  Doc comment inline (lines 63-66):
  > "Drain the set before awaiting children. An accept task may already be
  > waiting to register a just-accepted connection; leaving the mutex held
  > while joining would deadlock that task. Once cancelled, any late
  > registration observes cancellation and is rejected."
  Chapter: 4 — good concrete example of a subtle concurrency bug class
  (lock-held-across-await deadlock) and how the code avoids it.

- **`RuntimeTaskScope::task_count`** — lines 76-79, `#[cfg(test)]` only,
  test helper.

- **`log_task_completion`** — lines 82-88, free function; logs a spawned
  task's `JoinError` unless it was a plain cancellation.

- **`ServerRuntime` struct** — lines 90-120. Holds `tasks: Arc<RuntimeTaskScope>`
  (line 119) alongside all the shared server state (sessions, swarm_state,
  event buses, etc. — see fields list). One sentence: the whole-server
  handle that every connection-handling task clones to reach shared state;
  demonstrates the `Arc<RwLock<...>>` pattern relevant to Chapter 5.
  Chapter: 4 (owns the scope), 5 (shared-state registries — note nearly
  every field is `Arc<RwLock<_>>` or `Arc<Mutex<_>>`, e.g. `sessions:
  Arc<RwLock<HashMap<String, Arc<Mutex<Agent>>>>>` at line 92).

- **`ServerRuntime::spawn_main_accept_loop`** — lines 156-185. Spawns the
  TCP/local accept loop directly via `tokio::spawn` (NOT through
  `RuntimeTaskScope::spawn` — note this nuance: the *loop itself* bypasses
  the scope and gets its own `child_token()`, lines 157-158, while
  connections *it* accepts go through `spawn_client_task`, which does use
  the scope). Uses `tokio::select!` (lines 164-167) racing accept() against
  `cancellation.cancelled()`.
  **[RUST]** *plain English: `tokio::select!` runs several async operations
  concurrently and proceeds with whichever finishes first, cancelling the
  rest — used everywhere in this codebase as the low-level building block
  for "race an operation against a cancellation/timeout signal."*
  Chapter: 4.

- **`ServerRuntime::spawn_debug_accept_loop`** — lines 187-219, same pattern
  for the debug listener.

- **`ServerRuntime::spawn_gateway_accept_loop`** — lines 221-250, `async fn`
  returning `bool`, goes through `self.tasks.spawn(...)` (scoped).

- **`ServerRuntime::spawn_background_task`** — lines 252-264. Generic helper:
  wraps an arbitrary future in a `tokio::select!` against cancellation and
  registers it with the scope. This is the general-purpose "fire a
  cancellable background job" entry point other server code calls.
  Chapter: 4.

- **`ServerRuntime::spawn_client_task`** (266-280), **`spawn_gateway_client_task`**
  (282-296), **`spawn_debug_client_task`** (298-307) — all thin wrappers that
  clone `self` and register a per-connection handler via `self.tasks.spawn`.

- **`ServerRuntime::shutdown`** — lines 309-311, just delegates to
  `self.tasks.shutdown().await`.

- **`ServerRuntime::run_client_stream`** — lines 333-391, and
  **`run_debug_stream`** — lines 393-438: the actual per-connection body,
  each racing the real work against `cancellation.cancelled()` via
  `tokio::select!` (lines 376-379, 432-435) — i.e., cancellation is
  cooperative: the task must explicitly check/race the token, nothing
  preempts it.
  **[RUST]** *plain English worth flagging in the chapter: Rust/tokio
  cancellation is cooperative, not preemptive — a task keeps running until
  it hits an `.await` point that's racing the token. This is the same model
  as `AbortSignal` in JS/TS (code must check `signal.aborted` or pass the
  signal to an abortable API); it is not like killing a thread.*

- **Tests module** — lines 441-484, `runtime_task_scope_cancels_and_joins_owned_tasks`:
  demonstrates the scope's cancel+join contract end to end (spawns a task
  holding a `DropFlag`, cancels, asserts the task was dropped/joined, and
  that a post-shutdown spawn attempt is rejected). Good source for a
  Chapter 4 "how do we know this works" callout.

### `crates/jcode-agent-runtime/src/lib.rs` (283 lines total — full file read)

Confirmed present, 283 lines.

- **`InterruptSignal` struct** — lines 32-40.
  ```rust
  #[derive(Clone)]
  pub struct InterruptSignal {
      flag: Arc<std::sync::atomic::AtomicBool>,
      epoch: Arc<std::sync::atomic::AtomicU64>,
      notify: Arc<tokio::sync::Notify>,
  }
  ```
  Doc comment (lines 30-31): "Async-aware interrupt signal that combines
  AtomicBool (sync read) with tokio::Notify (async wake). Eliminates
  spin-loops during tool execution." One sentence: a cooperative-cancellation
  flag, similar in spirit to `runtime.rs`'s `CancellationToken`, but
  hand-rolled and specifically shaped for the agent's cancel/interrupt path
  (Esc/Ctrl+C), with an epoch counter to make repeated fire/reset races
  safe.
  **[RUST]** *plain English: an `AtomicBool` is a boolean that can be safely
  read/written from multiple threads at once without a lock (cheap,
  hardware-supported). `tokio::sync::Notify` is a low-level async
  wake-one-or-all-waiters primitive — think of it as a condition variable
  for async code: tasks can `.await` on `notified()` and be woken when
  someone calls `notify_waiters()`. Combining both means: cheap synchronous
  polling (`is_set()`) for hot paths, PLUS the ability to `.await`
  asynchronously without busy-looping, for cold paths.*
  Chapter: 4 (this is the `InterruptSignal` cancellation primitive named in
  the plan).

- **`InterruptSignal::new`** (43-49), **`fire`** (51-55), **`is_set`**
  (57-59), **`reset`** (61-63), **`epoch`** (68-70), **`reset_if_epoch`**
  (78-90), **`notified`** (92-106), **`as_atomic`** (108-110),
  **`same_instance`** (115-117).
  - `fire()` (51-55): bumps the epoch counter, sets the flag, then calls
    `self.notify.notify_waiters()` — i.e. it always does all three, so
    synchronous pollers (`is_set()`) and async waiters (`notified()`) both
    observe a fire regardless of which style of caller they are.
  - `reset_if_epoch(epoch)` (78-90) is the most interesting method design-
    wise: it only clears the flag if no *newer* fire has happened since the
    caller captured `epoch`; if a fire races in between the epoch check and
    the flag write, it re-sets the flag and re-notifies rather than losing
    that newer cancel. Doc comment (72-77):
    > "Reset the signal only if no newer `fire` happened since `epoch` was
    > captured... If a racing fire lands between the epoch check and the
    > reset, the fire is restored (flag re-set and waiters re-notified) so
    > no cancel is ever silently erased."
    Struct field doc comment (35-37) gives the "why" origin story: "Lets
    owners of a timed/deferred reset detect that a *newer* fire landed in
    the meantime and skip the reset instead of erasing a cancel the target
    has not observed yet (issue #428)."
  - `notified()` (92-106) has its own inline design-rationale comment
    (94-100) about why it calls `.enable()` explicitly on the `Notified`
    future before checking the flag — guards against a lost-wakeup race
    (issue #428) rather than relying on a tokio version-specific guarantee.
  This whole file is an unusually good "why is this so careful" exhibit for
  Chapter 4: it's solving the classic **lost-wakeup race** in cancellation
  signaling (a task fires cancel; a waiter is *between* checking the flag
  and registering to be woken; naive code drops the cancel). Both the
  struct-field comments and the 5 tests (lines 142-283) exist specifically
  to pin down and regression-test this race (referenced throughout as
  "issue #428").
  **[RUST]** *plain English for the handbook's TS analogy: this whole file
  is solving a problem that `AbortController`/`AbortSignal` in JS mostly
  sidesteps because JS is single-threaded — there's no race between "check
  if aborted" and "register a listener" the way there is when multiple OS
  threads can run Rust code truly in parallel. Worth flagging explicitly:
  this is one of the few places in the codebase where the Rust version is
  meaningfully more complex than the "obvious" JS/TS translation would be,
  specifically because of true multi-threading.*

- **Tests module** — lines 142-283, five tests, each targeting one specific
  race/guarantee:
  - `notified_future_receives_notify_waiters_from_creation` (153-173) —
    documents the tokio `Notify` creation-time-registration guarantee this
    code relies on.
  - `fire_never_loses_wakeup_while_notified_races` (180-207) — a 2000-
    iteration hammer test spawning a real multi-threaded runtime
    (`worker_threads(2)`, line 183) to catch the lost-wakeup race.
  - `notified_returns_immediately_when_already_fired` (210-217).
  - `reset_clears_fired_state` (220-229).
  - `reset_if_epoch_skips_when_newer_fire_landed` (235-259) and
    `reset_if_epoch_never_erases_concurrent_fire` (263-282) — directly test
    the epoch-guarded reset design described above.
  Chapter: 4 — strong material for a "how do you test concurrency code"
  sidebar.

- **Other types in this file, not concurrency-primary but worth knowing
  exist**: `SoftInterruptMessage`/`SoftInterruptSource` (4-18),
  `SoftInterruptQueue` type alias (21, `Arc<std::sync::Mutex<Vec<...>>>`),
  `BackgroundToolSignal`/`GracefulShutdownSignal` type aliases (25, 28, both
  bare `Arc<AtomicBool>` — simpler cousins of `InterruptSignal` without the
  async-notify piece), `StreamError` (128-140, unrelated — provider
  streaming error type, not a concurrency primitive; don't cite this file
  for it without flagging it's orthogonal to the cancellation story).

---

## Chapter 5 — Shared state without locks-as-a-crutch

No single file "owns" this chapter; it's a cross-cutting theme synthesized
from structures already cited above under Chapters 3/4/7. This section
pulls the relevant pieces together with their exact citations so Phase 2
doesn't have to re-derive them.

### The core nuance the plan asks for, precisely located

`docs/SWARM_ARCHITECTURE.md:307-311` ("Conflict Handling (No Locks)")
states "The system is optimistic by default (no locks)... Coordination
happens via DM or channel, not through the coordinator" — this is a claim
about the **agent-facing coordination protocol** (no agent blocks on a
mutex to edit a file or claim a task; conflicts are resolved by convention/
communication between agents, not mutual exclusion enforced by the
server). It is **not** a claim that the Rust implementation avoids
`std`/`tokio` locks internally — it very much does not:

- `crates/jcode-app-core/src/server/runtime.rs:90-120` — the `ServerRuntime`
  struct: nearly every field is `Arc<RwLock<_>>` or `Arc<Mutex<_>>`, e.g.
  `sessions: Arc<RwLock<HashMap<String, Arc<Mutex<Agent>>>>>` (line 92,
  double-locked: the registry itself behind a `RwLock`, each `Agent` behind
  its own `Mutex`), `is_processing: Arc<RwLock<bool>>` (95),
  `client_connections: Arc<RwLock<HashMap<...>>>` (98), `shared_context:
  Arc<RwLock<HashMap<...>>>` (100), `shutdown_signals:
  Arc<RwLock<HashMap<String, InterruptSignal>>>` (115).
- `crates/jcode-app-core/src/server/state.rs:108-113` — **`SwarmState`
  struct**, the shared handle passed around `swarm.rs`/`comm_session.rs`/
  `comm_await.rs`:
  ```rust
  #[derive(Clone)]
  pub struct SwarmState {
      pub members: Arc<RwLock<HashMap<String, SwarmMember>>>,
      pub swarms_by_id: Arc<RwLock<HashMap<String, HashSet<String>>>>,
      pub plans: Arc<RwLock<HashMap<String, VersionedPlan>>>,
      pub coordinators: Arc<RwLock<HashMap<String, String>>>,
  }
  ```
  Doc comment (106): "Shared ownership of the core persisted swarm
  coordination state." This is **the** concrete "shared state" struct for
  Chapter 5 — four independently-locked registries (members, swarm
  membership index, plans, coordinator-slot assignment), cloned cheaply
  (it's just `Arc` clones, `#[derive(Clone)]`) into every function that
  needs swarm state (seen throughout `swarm.rs`'s function signatures,
  e.g. `swarm_members: &Arc<RwLock<HashMap<String, SwarmMember>>>`
  parameters everywhere).
- `crates/jcode-app-core/src/server/state.rs:186-233`+ — **`SwarmMember`
  struct** (the live, in-memory member record — see the "follow-up
  resolved" note under the `jcode-swarm-core` section above for the
  `SwarmMember` vs. `SwarmMemberRecord` distinction: live struct uses bare
  `String` for `status`/`role`, persisted record uses typed enums).
- `crates/jcode-app-core/src/server/state.rs:15-38` — the process-global
  `BACKGROUND_TOOL_SIGNALS` registry (`static ...: LazyLock<StdMutex<HashMap<String,
  InterruptSignal>>>`, line 30) is a **second, independent** example of the
  "process-wide singleton behind a std `Mutex`" pattern also seen in
  `jcode-base/src/bus.rs`'s `Bus::global()` (`OnceLock`) and `swarm.rs`'s
  `pending_swarm_status_broadcasts()` (`OnceLock<StdMutex<HashMap<...>>>>`,
  `swarm.rs:108-113`). Its doc comment (15-29) is an unusually clear
  worked example of *why* a lock-free fallback path needed its own
  side-registry:
  > "The background-tool... signal lives on the `Agent`, so a
  > `SessionControlHandle` can normally only obtain it by locking the agent
  > mutex. When a turn is busy..., `refresh_session_control_handle` falls
  > back to a lock-free `cancel_only` handle that historically dropped the
  > background signal entirely, which made Alt+B/Ctrl+B silently no-op...
  > This registry is populated every time a full `SessionControlHandle` is
  > built..., so the lock-free fallback can still fire the background
  > signal without the agent lock."
  This is a strong Chapter 5 exhibit: a real bug (a UI hotkey silently
  no-op'd) caused by *not* having a lock-free path, fixed by adding a
  small separately-locked side-index rather than widening the main lock's
  scope — i.e. "avoid locks" in this codebase in practice often means
  "narrow what's behind each lock and add small independent locks," not
  "have zero locks."

### The "debounce instead of lock harder" pattern (recurs at least twice)

Two independent, not-code-shared implementations of the same idea —
coalesce a burst of rapid state changes into one delayed broadcast rather
than serializing writers through a lock or sending N redundant messages:
1. `crates/jcode-app-core/src/server/swarm.rs:87-231` (approx) —
   `PendingSwarmStatusBroadcast`/`pending_swarm_status_broadcasts()`/
   `broadcast_swarm_status` (742-812): debounces member-status broadcasts
   over `swarm_status_debounce_ms()` (default configurable) once swarm size
   passes `swarm_status_debounce_member_threshold()` (default 2).
2. `crates/jcode-base/src/bus.rs:542-606` — `Bus::publish_models_updated`:
   debounces `BusEvent::ModelsUpdated` over a fixed `MODELS_UPDATED_DEBOUNCE`
   = 750ms (line 476).
Both are good, compact, fully-read (see citations above) code samples for
a "how do you avoid a broadcast storm without a lock" handbook callout.

### Cross-reference for Chapter 5's TS analogy

**[RUST]** *plain English, for the whole chapter: `Arc<RwLock<T>>` is
Rust's "shared, many-readers-or-one-writer" container — `Arc` (atomic
reference count) lets multiple owners hold the same data, `RwLock` lets
many concurrent readers OR one exclusive writer in at a time (unlike
`Mutex`, which is always exclusive even for reads). None of this exists in
single-threaded JS/TS — a `Map`/object is inherently only ever touched by
one turn of the event loop at a time, so "shared mutable state across
concurrent workers" in a TS translation instead means either (a) a single
process holding the state with async functions that never truly run
literally simultaneously (cooperative concurrency — no lock needed, just
careful `await` placement to avoid interleaving bugs), or (b) genuinely
parallel `worker_threads`, which would need `SharedArrayBuffer` +
`Atomics` for true shared memory, or (far more commonly) message-passing
to a single owning thread instead of shared mutable structures at all.
This is a good moment in the handbook to explain that Rust's `Arc<RwLock<>>`
proliferation isn't over-engineering — the app genuinely runs many OS
threads via tokio's multi-threaded runtime, so without it there would be
data races, not just theoretical ones.*

---

## Verification / broken-links checklist

Every file in the task brief plus every file this map additionally opened
to resolve a question, checked via `wc -l` and/or `Read` on 2026-08-12
against `master` @ `5ae238574`. **Nothing in the task brief's file list
was missing or unresolvable.** Status of each:

| # | Path | Status | Lines |
| - | ---- | ------ | ----- |
| 1 | `crates/jcode-app-core/src/server/runtime.rs` | OK, fully read | 484 |
| 2 | `crates/jcode-app-core/src/server/swarm.rs` | OK, real code read 1-1725; 1726-3170 is tests (do not cite as prod behavior) | 3170 |
| 3 | `crates/jcode-app-core/src/server/comm_session.rs` | OK, key functions read; some helper fns only grep-located (flagged inline) | 1434 |
| 4 | `crates/jcode-app-core/src/server/comm_await.rs` | OK, fully read | 680 |
| 5 | `crates/jcode-app-core/src/tool/batch.rs` | OK, fully read (371/371 lines) | 371 |
| 6 | `crates/jcode-base/src/bus.rs` | OK, `BusEvent`/`Bus` fully read; payload structs grep-located only | 642 |
| 7 | `crates/jcode-agent-runtime/src/lib.rs` | OK, fully read including all 5 tests | 283 |
| 8 | `crates/jcode-swarm-core/src/lib.rs` | OK, fully read; **re-scoped away from Chapter 6, see CRITICAL FINDING** | 837 |
| 9 | `crates/jcode-task-types/src/lib.rs` | OK, exists, but **confirmed irrelevant to swarm/DAG** (goal/todo tracking crate) — symbol list only, not deeply read since it's out of scope | 854 |
| 10 | `crates/jcode-plan/src/dag/mod.rs` | OK, fully read — **confirmed as the real DAG engine core, primary Chapter 6 source** | 682 |
| 10b | `crates/jcode-plan/src/dag/ops.rs` | OK, exists; signatures + doc comments grepped/read, function bodies NOT individually re-read line by line — re-verify exact body text before quoting code blocks (not just signatures) in Phase 2/3 | 878 |
| 10c | `crates/jcode-plan/src/dag/schedule.rs` | OK, fully read | 106 |
| 10d | `crates/jcode-plan/src/dag/sim.rs` | OK, exists, NOT read (only referenced) — open before citing | 155 |
| 10e | `crates/jcode-plan/src/dag/tests.rs` | OK, exists, NOT read (test suite, out of scope for prod-behavior citations) | 1392 |
| 10f | `crates/jcode-plan/src/bridge.rs` | OK, header doc comment + top-level fn list read; not every fn body opened | 503 |
| 10g | `crates/jcode-plan/src/lib.rs` (`VersionedPlan`/`PlanItem`) | OK, exists; only symbol list grepped, field lists NOT verified — **do not claim specific `PlanItem`/`VersionedPlan` field names/types beyond `id`/`content`/`status` (used in swarm.rs citations above) without opening this file directly** | 1201 |
| 11a | `docs/SWARM_ARCHITECTURE.md` | OK, fully read | 318 |
| 11b | `docs/SWARM_TASK_GRAPH.md` | OK, fully read | 605 |
| extra | `crates/jcode-config-types/src/lib.rs` (`SwarmSpawnMode`) | OK, relevant section read (opened to resolve Chapter 2's "mode-gated spawning") | n/a (large file, only lines ~630-660 read) |
| extra | `crates/jcode-app-core/src/server/state.rs` (`SwarmState`, `SwarmMember`) | OK, relevant sections read (opened to resolve Chapter 5 shared-state material and the `SwarmMember`/`SwarmMemberRecord` discrepancy) | n/a (large file, only lines 1-233 read) |

### Known gaps for Phase 2 to close (do not treat map silence as "doesn't exist" — re-derive if needed)

1. **Live `swarm` tool schema/action names** — no tool-definition file
   (e.g. wherever the `swarm` tool's `parameters_schema()`/action enum
   lives) was opened in this pass. `SWARM_TASK_GRAPH.md` §8's "Proposed
   tool surface" and the prompt-text action names in
   `jcode-swarm-core::append_deep_node_instructions`/
   `append_deep_gate_instructions` are strong circumstantial evidence for
   current action names (`expand_node`, `complete_node`, `inject_gap`,
   `report`), but Chapter 2/6/7 should not assert the tool's exact JSON
   parameter shape without opening that file first.
2. **Worktree Manager implementation** — no source file for worktree
   management was in the task brief or opened in this pass;
   `SWARM_ARCHITECTURE.md`'s Worktree Usage section (153-192) is the only
   source, and it's prose/design-doc only, not source-verified.
3. **Reparenting-on-departure behavior** — `SWARM_ARCHITECTURE.md:40-45`
   claims mid-tree departures reparent children to their grandparent; not
   independently located in `swarm.rs::remove_session_from_swarm`
   (995-1219), which was identified but not read line-by-line in this
   pass (it's the largest function in the file). Open it fully before
   Chapter 2 asserts reparenting mechanics with a code citation.
4. **`jcode-plan/src/dag/ops.rs` function bodies** — signatures and doc
   comments were read for every `pub fn`, but full bodies were not
   individually verified line-by-line the way `mod.rs` and `schedule.rs`
   were. Safe to cite function existence/location/doc-comment rationale;
   re-open before quoting an actual code block from this file.
5. **`PlanItem`/`VersionedPlan` full field lists** (`jcode-plan/src/lib.rs`)
   — only confirmed to exist at the cited line numbers; fields beyond
   what's referenced elsewhere in this map (`id`, `content`, `status`,
   `blocked_by`, `assigned_to` — all seen via other files' usage, not this
   file directly) are unverified.
6. **`jcode-swarm-core`'s `SwarmRole`/`SwarmLifecycleStatus` vs. the live
   `SwarmMember.status: String`/`.role: String`** — confirmed these are
   different representations (see Chapter 5/7 sections above); the actual
   translation code between them (if any) was not located.

None of the above are contradictions found in the code — they are simply
citations this map did not chase to full ground truth within scope/budget,
flagged so Phase 2/3 agents re-derive rather than invent them.

---
