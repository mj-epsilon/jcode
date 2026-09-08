# Phase 2 Notes — Chapter 1: Why parallel agents, and why this codebase

Status: every citation below was opened and read directly by this Phase 2 pass
(not just trusted from the Phase 1 map) on 2026-08-12, master @ `5ae238574`
(or later — no commits landed during this session). Where I relied on the
map's earlier reading without re-opening the file myself, I say so explicitly.

---

## 1. The plain-English problem this chapter frames

Before any Rust: a single coding agent is a **serial loop** — read, think, act,
observe, repeat, one tool call at a time, on one working directory, one
conversation thread. That's fine for a scoped task. It breaks down for large,
breadth-first work: "explore this whole codebase," "fix these 40 independent
lint violations," "implement 6 unrelated features." Doing those serially in
one agent thread wastes the fact that most of that work has no dependency on
the rest of it — it's the same intuition as parallelizing a CI test suite
across shards instead of running every test in one process.

jcode's swarm system is the answer to "how do you let one coding agent become
many, working at once, without turning into an unmanageable mess of
concurrent state." The chapter should motivate this *before* showing any
Rust: the reader needs to feel the shape of the problem (fan-out work,
fan-in results, and prevent the resulting concurrency from corrupting shared
state) before any tokio primitive makes sense.

## 2. Why *this* codebase is a good teaching example

No actor framework, no distributed job queue — it's hand-rolled on top of
tokio's own primitives (`JoinSet`, `CancellationToken`, `broadcast`,
`RwLock`, `Mutex`, plain `tokio::spawn`). This is pedagogically useful:
every concurrency primitive used is a *named, well-known* Rust/tokio
building block rather than framework-specific magic, so each one has a
nameable TypeScript/Node analogue (worker_threads, Promise.all,
EventEmitter, AbortController) — exactly the mapping the handbook wants to
draw. Verified by directly reading `crates/jcode-app-core/src/server/runtime.rs`
(imports/struct at lines 1-37) and `crates/jcode-app-core/src/server/swarm.rs`
(imports at lines 1-20, `use futures::future::try_join_all;` at line 9,
`use tokio::sync::{Mutex, RwLock, broadcast};` at line 20) — no actor-model
crate (`actix`, `ractor`, etc.) is imported anywhere in the swarm code path.

## 3. Vocabulary (each term below is defined against code I opened myself)

### Agent / session
A **session** is the persistent identity + conversation state of one running
agent loop; an **agent** (`Agent` struct, imported at
`crates/jcode-app-core/src/server/swarm.rs:4`, `use crate::agent::Agent;`)
is the live in-memory object that drives it. Spawning a new swarm member
means creating a brand-new `Session` + `Agent` pair — literally, not
metaphorically:
```rust
// crates/jcode-app-core/src/server/swarm.rs:1548-1551 (inside run_swarm_task —
// see the "important caveat" in §6 below about this function's real reachability)
let mut session = Session::create(
    Some(session_id),
    Some(format!("{} (@{} swarm)", description, subagent_type)),
);
```
**[RUST plain-English]**: `Session::create` returns an owned struct value
(not a handle into someone else's registry) — the caller decides what to do
with it next (here: set fields, `.save()`, wrap in an `Agent`). Closest TS
mental model: `new Session(...)` — an ordinary constructor call, not a
factory registering itself anywhere yet.

The live, in-memory bookkeeping record for "a session that is part of a
swarm" is a separate struct, `SwarmMember`
(`crates/jcode-app-core/src/server/state.rs:186-233`, confirmed present via
the Phase 1 map and cross-checked by reading its construction site directly
at `crates/jcode-app-core/src/server/comm_session.rs:502-528`, the
`register_visible_spawned_member` function). Notable fields I personally
verified at that construction site: `session_id`, `event_tx` (a per-member
`mpsc::UnboundedSender`, line 508), `swarm_id`, `status`/`detail` (bare
`String`s), `report_back_to_session_id` (line 517 — this is the parent
edge, see §4), `role` (defaults to `"agent"`, line 519), `is_headless`
(line 522). So: **agent = the live Rust object executing turns; session =
its persisted identity/state; `SwarmMember` = the swarm's bookkeeping
record about that session** — three related but distinct things, worth
distinguishing explicitly for the reader since English "agent" is often used
loosely for all three in the design docs.

### Coordinator
A **role string** (`"agent"` or `"coordinator"` — confirmed as the only two
values the tool schema accepts, see
`crates/jcode-app-core/src/tool/communicate.rs:2005-2008`,
`"role": { "type": "string", "enum": ["agent", "coordinator"] }`) held by
exactly one `SwarmMember` per `swarm_id` at a time, tracked in
`SwarmState.coordinators: Arc<RwLock<HashMap<String, String>>>`
(`crates/jcode-app-core/src/server/state.rs:108-113`, structure confirmed by
the Phase 1 map and consistent with every `swarm_coordinators` parameter
I opened directly in `comm_session.rs`/`swarm.rs`). The coordinator slot is
narrowly scoped: it exists **only** to own the one shared `VersionedPlan`
per `swarm_id` (propose/approve/assign/task-control) — it is not the same
concept as "who owns this spawn subtree" (see `report_back_to_session_id`
below, which every spawning agent gets regardless of coordinator status).
I personally read the coordinator-claim logic in full at
`crates/jcode-app-core/src/server/comm_session.rs:1380-1427`
(inside `ensure_spawn_coordinator_swarm`): only a **root** session (no
`report_back_to_session_id`, i.e. not itself spawned by anyone) can claim
the slot, and only when it is empty or the current holder is "stale"
(disconnected/terminal — the staleness check itself is at lines 1294-1308,
worth a callout: it treats a coordinator whose every event channel is
closed as stale even without a clean status transition, explicitly to avoid
a wedged/dead coordinator blocking the slot forever).

### Swarm
Not a struct with its own lifecycle — a **`swarm_id` string** that a set of
`SwarmMember`s share. The grouping itself is just an index:
`SwarmState.swarms_by_id: Arc<RwLock<HashMap<String, HashSet<String>>>>`
(`state.rs:108-113`, `swarm_id -> set of member session_ids`). A `swarm_id`
is normally *derived from the working directory*, specifically the git
common dir, so that multiple git worktrees of the same repo resolve to the
same `swarm_id` — I opened and read this directly:
```rust
// crates/jcode-app-core/src/server/util.rs:329-341
pub(crate) fn swarm_id_for_dir(dir: Option<PathBuf>) -> Option<String> {
    if let Ok(sw_id) = std::env::var("JCODE_SWARM_ID") { ... }
    let dir = dir?;
    if let Some(git_common) = git_common_dir_for(&dir) {
        return Some(git_common.to_string_lossy().to_string());
    }
    Some(dir.to_string_lossy().to_string())
}
```
with `git_common_dir_for` (`util.rs:290-327`) explicitly walking up to a
`.git` file/dir and, when it finds a `<repo>/.git/worktrees/<name>` gitdir
(the on-disk shape `git worktree add` produces), resolving to the **shared**
common dir (`util.rs:315-321`) rather than the per-worktree one. This is
real, verified code confirming the design doc's claim that swarms span
worktrees of one repo. **Nuance** (also read directly,
`util.rs:343-357`, `swarm_id_for_session`): root sessions actually default
to *owning their own swarm* rather than sharing one derived purely from
directory, per its own doc comment — "Deriving that id from the working
directory made every session opened in one repository share one plan, even
when those sessions were unrelated. Root sessions therefore own a swarm by
default." `JCODE_SWARM_ID` remains an explicit opt-in for intentionally
sharing one swarm across independently-started root sessions. Worth
presenting both functions in Chapter 1 as "how big is a swarm, by default"
material, since it's genuinely two-layered (git-worktree awareness +
explicit-opt-in override).

### Worktree Manager — CLOSES MAP GAP #1 (verified, not invented)

**Finding: "Worktree Manager" is design vocabulary, not an implemented role
or type.** I searched the whole `crates/` tree for `WorktreeManager`,
`worktree_manager`, and any role/enum tied to worktree ownership. There is
no such struct, enum variant, or gate anywhere in the code. What I found
instead:

1. `docs/SWARM_ARCHITECTURE.md:71-75` describes it in prose only ("Owns a
   single worktree scope... Responsible for integration when that worktree
   scope is done.") with no corresponding Rust construct.
2. The only place the phrase "worktree manager" appears **inside source
   code** at all is a user-facing error string, which I opened and traced
   to its actual authorization check:
   ```rust
   // crates/jcode-app-core/src/server/comm_sync.rs:346-353
   if !can_read_full_context(&req_session_id, &target_session, swarm_members).await {
       let _ = client_event_tx.send(ServerEvent::Error {
           id,
           message: "Only the coordinator, worktree manager, or the target session may read full context. Use summary for lightweight access.".to_string(),
           ...
   ```
   But the function it's guarding, `can_read_full_context`
   (`comm_sync.rs:178-192`), only checks two things:
   ```rust
   // crates/jcode-app-core/src/server/comm_sync.rs:178-192
   async fn can_read_full_context(...) -> bool {
       if req_session_id == target_session { return true; }
       let members = swarm_members.read().await;
       members.get(req_session_id).map(|member| member.role == "coordinator").unwrap_or(false)
   }
   ```
   i.e. **self, or role == "coordinator" — nothing else.** There is no
   third branch for a worktree-manager role; the error message's mention of
   "worktree manager" is aspirational text left over from the design intent,
   not a real permission the code grants. Since `role` is a bare `String`
   restricted by the tool schema to `{"agent", "coordinator"}` (verified
   above), there is no third role value a session could even hold.
3. `docs/SWARM_TASK_GRAPH.md:29-30` (the newer, DAG-first design doc) states
   the project's own intended direction explicitly: **"Coordinator /
   worktree-manager roles demote to scheduler policy, not user-facing
   concepts."** — i.e. the design's own roadmap is to make this *not* a
   first-class role going forward, which is consistent with finding no
   implementation of it today.

**How Chapter 1 should present this, honestly**: introduce "Worktree
Manager" as a vocabulary term from the design docs describing an
*integration-ownership responsibility* an agent can informally hold when
work is split across git worktrees (via spawn prompt / convention), but
explicitly flag that — as of this reading — there is no dedicated Rust type,
enum variant, or authorization gate for it; the one place it's named in
source code (an error message) does not correspond to an actual check. This
is a legitimate, well-evidenced finding, not a gap to paper over.

## 4. `report_back_to_session_id` — the spawn tree, previewed for Ch1

(Full mechanics belong in Chapter 2; Chapter 1 only needs the vocabulary
and the "why no separate tree structure" rationale.) I re-read this
directly:
```rust
// crates/jcode-app-core/src/server/swarm.rs:34-40 (doc comment) + 41-60 (body)
/// The spawner/parent edge is encoded by `report_back_to_session_id`: a child
/// spawned by `P` reports back to `P`. Walking that chain reconstructs the spawn
/// tree without persisting a separate parent field. Cycles (which should never
/// happen) are guarded against with a visited set.
pub(super) fn swarm_ancestors(
    members: &HashMap<String, SwarmMember>,
    session_id: &str,
) -> Vec<String> { ... }
```
Good one-line framing for Chapter 1: *the spawn tree isn't a separate data
structure — it's a derived view, recomputed on demand from one field each
member already carries.* This is a nice small design-pattern callout:
"store the minimal edge, derive the tree" vs. "maintain a redundant tree
structure that could drift out of sync."

## 5. Architecture diagram — spec for Phase 3 (not drawn here)

Suggested single diagram for Chapter 1, with every box backed by a citation
above or in notes-ch02.md:

- **Top layer — process/runtime**: `ServerRuntime` (owns everything;
  `crates/jcode-app-core/src/server/runtime.rs:90-120`, per the Phase 1 map,
  not re-opened line-by-line by me this pass but consistent with every
  `&Arc<RwLock<...>>` parameter I personally saw threaded through
  `swarm.rs`/`comm_session.rs`) with `RuntimeTaskScope`
  (`runtime.rs:33-37`) as its structured-concurrency shutdown boundary —
  belongs more to Chapter 4, but worth a small box in the Ch1 diagram as
  "the thing that owns every task."
- **Shared state layer**: `SwarmState { members, swarms_by_id, plans,
  coordinators }` (`state.rs:108-113`) — four independently-locked
  registries, cloned cheaply (`Arc` clone) into whichever handler needs
  them.
- **Swarm layer**: one box per `swarm_id`, containing a tree of
  `SwarmMember`s connected by `report_back_to_session_id` edges, with
  exactly one (optional) coordinator-slot pointer into that tree, per
  §3/§4 above.
- **Agent-facing interface layer**: the `swarm` tool (`name()` returns
  `"swarm"`, `crates/jcode-app-core/src/tool/communicate.rs:1940-1942`,
  verified directly) — this is the *only* surface a live LLM agent uses to
  spawn/message/await; tool execution sends a `Request` enum value over an
  internal channel to a central handler rather than mutating swarm state
  directly (see notes-ch02.md §2 for the exact spawn dispatch).
- **Cross-cutting layer**: the global event bus,
  `Bus::global()` (`crates/jcode-base/src/bus.rs`, per the Phase 1 map,
  singleton via `OnceLock`) — not re-verified line-by-line by me this pass,
  flagged here only as a diagram box, full detail belongs to Chapter 7.

TS-equivalent framing note for Phase 3 (mapping only, not code): the whole
diagram is "one Node process, N `worker_threads` (or just N concurrently
running async tasks if modeling headless/inline spawns, which run
in-process) registered in a parent-side `Map<id, WorkerHandle>`, with a
single shared event emitter for cross-cutting notifications." Emphasize
that jcode's default spawn modes (`Headless`/`Inline`, see notes-ch02.md)
run the child **in the same OS process** as an async task, not a literal
OS thread/subprocess — so the more accurate TS analogue for the *default*
path is spawning another concurrently-running `async` job in the same
process (e.g. an unawaited promise tracked in a map), not literally a
`worker_thread`; `worker_threads` becomes the right analogy specifically
for jcode's `Visible` spawn mode, which does launch a separate OS-level
terminal/process.

## 6. Open questions / things NOT to assert without further verification

- I did not re-open `crates/jcode-app-core/src/server/runtime.rs` or
  `crates/jcode-base/src/bus.rs` line-by-line myself this pass (I relied on
  the Phase 1 map, which did read them fully, per its own verification
  table). Chapter 1 can cite them via the map's line numbers, but a later
  phase should treat those two files as "map-verified, not
  independently re-verified in Phase 2" if it wants the strictest
  chain of custody.
- **Important correction that also affects how Chapter 1 should describe
  "the" fan-out/fan-in flow**: `run_swarm_task`/`run_swarm_message`
  (`swarm.rs:1528-1701`, the `try_join_all` planner→fan-out→fan-in
  function the original plan calls "the flagship pattern") is **not**
  reachable from the live `swarm` tool at all. I traced its only two
  callers and both are debug/dev-tooling entry points, not the agent-facing
  tool surface:
  ```rust
  // crates/jcode-app-core/src/server/debug_command_exec.rs:132-139
  if trimmed.starts_with("swarm_message:") {
      ...
      let final_text = super::run_swarm_message(agent.clone(), msg).await?;
      return Ok(final_text);
  }
  ```
  ```rust
  // crates/jcode-app-core/src/server/debug_jobs.rs:77-95
  if trimmed.starts_with("swarm_message_async:") {
      ...
      let result = super::run_swarm_message(agent.clone(), &msg).await;
  ```
  Both are string-prefixed debug-socket commands (`swarm_message:`,
  `swarm_message_async:`), not anything a spawned agent's `swarm` tool call
  can trigger — the `swarm` tool's own `"message"` action is a plain
  DM/broadcast/channel router (`communicate.rs:2261-2276`, read directly:
  "`message` is the general-purpose send: it routes by the fields
  provided... With `to_session` it acts as a DM, with `channel` it posts to
  that channel, and with neither it broadcasts") — it does **not** call
  `run_swarm_message`. **Chapter 1 (and especially Chapter 2/3) should not
  present `try_join_all` planner→fan-out→fan-in as "what happens when an
  agent calls the swarm tool."** It's real, working code, but it's a
  separate, debug-tooling-only orchestration mode. This is flagged in more
  detail in notes-ch02.md since it directly affects "anatomy of a spawn."
  Full detail belongs to whichever agent writes Chapter 3, but Chapter 1's
  framing/diagram should not imply this is the primary path either.
