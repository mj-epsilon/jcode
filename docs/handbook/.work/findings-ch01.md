# Findings — Chapter 1 (draft-ch01.md)

Reviewed adversarially against source at HEAD (`5ae238574` or later, no
commits landed during this session). Every inline citation and every
uncited-but-checkable factual/API claim was traced to source directly.

## Citations checked

| # | Citation in draft | Verdict | Notes |
|---|---|---|---|
| 1 | `swarm.rs:1-20` — `use futures::future::try_join_all;` (line 9), `use tokio::sync::{Mutex, RwLock, broadcast};` (line 20) | CONFIRMED | Exact match, including line numbers for both `use` statements. |
| 2 | `swarm.rs:1548-1551` — `Session::create(Some(session_id), Some(format!(...)))` | CONFIRMED | Exact match, verbatim. `Session::create` signature (`crates/jcode-base/src/session.rs:775`, `pub fn create(parent_id: Option<String>, title: Option<String>) -> Self`) matches the two-`Option` shape shown. |
| 3 | `comm_session.rs:502-528` — "register_visible_spawned_member — constructs the SwarmMember record" | CONFIRMED | Line 502 is the `{` opening the `swarm_members.write().await` block, 528 is its closing `}`; the range is exactly the `SwarmMember { ... }` construction. Function name is correct. Minor nuance not flagged in the draft: this is the *visible*-spawn-mode registration path specifically (other spawn modes construct `SwarmMember` elsewhere, e.g. `headless.rs:242`, `client_session.rs:365`) — but the draft never claims this is the *only* construction site, so this isn't a hallucination, just worth a revision agent knowing about if extra precision is wanted. |
| 4 | `tool/communicate.rs:2005-2008` — `"role": { "type": "string", "enum": ["agent", "coordinator"] }` | CONFIRMED | Exact match at those lines. |
| 5 | `server/util.rs:329-341` — `swarm_id_for_dir` | CONFIRMED | Function body matches verbatim, including the `JCODE_SWARM_ID` env check and git-common-dir fallback. |
| 6 | `server/swarm.rs:34-40` (doc comment on `swarm_ancestors`) | CONFIRMED | Exact text match, line-for-line. |
| 7 | `server/comm_sync.rs:346-353` — the "worktree manager" error message | CONFIRMED | Exact text and exact line range (346 = `if !can_read_full_context`, 353 = closing `}`). |
| 8 | `server/comm_sync.rs:178-192` — `can_read_full_context` | CONFIRMED | Exact match, both the code and the line range (178 = `async fn`, 192 = closing `}`). |

## Uncited narrative claims spot-checked against source anyway

- "At most one member per swarm holds the coordinator role... only a root
  session... is allowed to claim the slot, and only when it's empty or the
  current holder looks stale (every one of its event channels closed, even
  without a clean shutdown)" — CONFIRMED against
  `comm_session.rs:1234-1330` (`ensure_spawn_coordinator_swarm`, staleness
  check at ~1294-1305: `unreachable = member.event_tx.is_closed() &&
  member.event_txs.values().all(|tx| tx.is_closed())`). Not cited inline in
  the draft, but accurate.
- `SwarmState { members, swarms_by_id, plans, coordinators }` "four
  independently-locked registries" (architecture diagram) — CONFIRMED
  verbatim against `state.rs:108-113`.
- `role: String` "It's literally a string" — CONFIRMED against
  `state.rs:216` (`pub role: String`, comment `Role: "agent" or
  "coordinator"`).
- The `swarm` tool is "the ONLY way a live LLM agent spawns/messages/awaits
  others" and its `spawn` action is a distinct code path from
  `run_swarm_task`/`run_swarm_message` — CONFIRMED. `run_swarm_message`'s
  only two callers are `debug_command_exec.rs:138` and
  `debug_jobs.rs:95`, both debug-socket string-command handlers, not the
  live tool dispatch (`communicate.rs:2720`, `"spawn" =>` branch, which
  calls into `spawn_swarm_agent` in `comm_session.rs`, the subject of
  Chapter 2). This is a genuinely important and correct claim — it heads
  off a real trap (mistaking `run_swarm_task`'s `try_join_all` shape for
  the live spawn path).
- Doc quotes: `docs/SWARM_ARCHITECTURE.md` "Worktree Manager" section
  ("Owns a single worktree scope"..."Responsible for integration when that
  worktree scope is done") — CONFIRMED verbatim at lines 71-75.
  `docs/SWARM_TASK_GRAPH.md` "Coordinator / worktree-manager roles demote
  to **scheduler policy**, not user-facing concepts" — CONFIRMED verbatim
  at line 32.
- The "Worktree Manager is not backed by code" claim itself — CONFIRMED by
  independently grepping for `WorktreeManager`/`worktree_manager` across
  `crates/`: no struct, enum, or gate found; the phrase appears in source
  only inside the one error string already cited.

## TypeScript code blocks

1. `Session` class (create/constructor) — honestly labeled `// idiomatic TS
   equivalent — this code does not exist in jcode`. Correct usage of
   `crypto.randomUUID()` (standard global in Node 19+/browsers). No issues.
2. `tryClaimCoordinator` function — honestly labeled the same way. Logic is
   internally consistent with the Rust behavior it's illustrating (root-only
   claim, staleness override) and uses only plain `Map`/`boolean` — no
   fabricated APIs. No issues.

## Narrative-vs-Phase-2-notes divergence check

Compared every confident claim in the draft against `notes-ch01.md`'s "open
questions" section (§6). The draft's closing caveat
(`run_swarm_task`/`try_join_all` is debug-only, not the live spawn path)
directly reflects the notes' flagged correction rather than glossing over
it — this is the correct behavior for the anti-hallucination gate, not a
violation of it. No instance found where the draft asserts something as
settled fact that the notes had flagged as open/unverified.

## Verdict

Zero MISCITED or UNVERIFIABLE findings. Chapter 1 is clean — every citation
resolves exactly to the claimed line range and content, every uncited
factual claim checked holds up, both TS blocks are honestly labeled and
technically sound, and the chapter's most delicate claim (the "Worktree
Manager isn't real code" and "try_join_all isn't the live path" findings)
are both independently verified as accurate rather than invented.
