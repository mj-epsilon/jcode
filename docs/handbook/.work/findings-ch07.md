# Findings — Chapter 7 (Communication Topology)

Reviewed against source directly on 2026-08-12. This is the most heavily
and most accurately cited of the four chapters reviewed — nearly every
citation is pixel-exact. One minor range-boundary finding below; everything
else CONFIRMED.

## Finding

### MISCITED (minor) — `client_comm_message.rs:251-272` for the channel-handling block

Draft cites this range for the `} else if let Some(ref channel_name) =
channel { ... }` snippet in the "Channels" section. Actual source:

```
251: let target_sessions: Vec<String> = if let Some(target) = resolved_to_session {
252:     vec![target]
253: } else if let Some(ref channel_name) = channel {      <- quoted block actually starts here
...
269:     }                                                  <- quoted block actually ends here
270: } else {
271:     subtree_broadcast_targets
272: };
```

The cited range (251-272) does fully *contain* the quoted code, but it also
pads in 2 lines belonging to the DM branch (251-252, `if let Some(target) =
resolved_to_session { vec![target]`) at the front and 3 lines belonging to
the final broadcast-fallback branch (270-272, `} else { subtree_broadcast_targets };`)
at the end — neither of which is part of what's being described as "the
channels code." Every other multi-line citation in this chapter is
boundary-exact (verified below), which makes this one stand out as
imprecise by comparison, though it's not misleading about content (nothing
quoted is fabricated).

**Correct citation: `client_comm_message.rs:253-269`.**

## Everything else: CONFIRMED

| Citation | Claim | Verdict |
| --- | --- | --- |
| `client_comm_message.rs:217-223` | `scope` routing decision (dm/channel/broadcast) | CONFIRMED — exact match |
| `client_comm_message.rs:35-80` | `resolve_dm_target_session` | CONFIRMED — function spans exactly 35-80 (closing brace at 80) |
| `client_comm_message.rs:230-249` | subtree-broadcast targets block, incl. comment | CONFIRMED — exact match, comment and code both verbatim |
| `state.rs:204,218` | `SwarmMember.status: String` / `.role: String` with their doc comments | CONFIRMED — line 204 is `pub status: String,` under "Lifecycle status (ready, running, completed, failed, stopped, etc.)"; line 218 is `pub role: String,` under "Role: \"agent\" or \"coordinator\"" — exact match on both |
| `comm_sync.rs:144-176` | `ensure_same_swarm_access` — shared-`swarm_id` gate | CONFIRMED — function spans exactly 144-176 |
| `comm_sync.rs:249-324` | `handle_comm_status` — lock-free snapshot, `try_lock` for provider/model, degrades to `(None, None)` | CONFIRMED — function spans exactly 249-324; `try_lock()` and `(None, None)` fallback confirmed verbatim at lines 294-298 |
| `comm_sync.rs:194-243` | `handle_comm_summary` — `try_lock`, default limit 10, explicit busy error with `retry_after_secs: Some(1)` | CONFIRMED — function spans exactly 194-243; error string `"Session '{}' is busy; try summary again shortly"` matches verbatim; `retry_after_secs: Some(1)` confirmed |
| `comm_sync.rs:326-382` | `handle_comm_read_context` — gated by `can_read_full_context`, returns `agent.get_history()` | CONFIRMED — function spans exactly 326-382; error string "Only the coordinator, worktree manager, or the target session may read full context. Use summary for lightweight access." matches verbatim |
| `comm_sync.rs:178-192` | `can_read_full_context` — self or `role == "coordinator"` bare-string check | CONFIRMED — function spans exactly 178-192, exact logic match |
| `bus.rs:499-502` | `Bus::global()` singleton | CONFIRMED — exact match (same as Chapter 5's citation of the same function) |
| `bus.rs:525-533` | `Bus::publish` / `let _ = self.sender.send(event);` | CONFIRMED — function spans exactly 525-533; the quoted line is line 532, within range |
| `swarm.rs:685-970` | range containing `broadcast_swarm_status_now`, `broadcast_swarm_status`, `broadcast_swarm_plan_with_previous`, `send_swarm_plan_to_session` | CONFIRMED — `broadcast_swarm_status_now` starts exactly at 685, `send_swarm_plan_to_session` (the last function in the group) closes exactly at 970 |
| `state.rs:243-258` | `SwarmMember::durable_record` | CONFIRMED — exact match, function spans exactly 243-258 |
| `jcode-swarm-core/src/lib.rs:107-115` | `impl From<String> for SwarmRole` | CONFIRMED — exact match, spans exactly 107-115 |
| `jcode-swarm-core/src/lib.rs:174-193` | `impl From<String> for SwarmLifecycleStatus`, "13 named variants plus `Other(String)`" | CONFIRMED — exact match, spans exactly 174-193; variant count independently verified: Spawned/Ready/Running/RunningStale/Completed/Done/Failed/Stopped/Crashed/Queued/Blocked/Pending/Todo = exactly 13 named variants + `Other` |
| `swarm_persistence.rs:341-390` | `recover_member_status` — Running→Crashed, Ready→Stopped, headless-non-terminal→Crashed with Completed/Done/Failed/Stopped carve-out | CONFIRMED — function spans exactly 341-390, all three rewrite rules match precisely, including the carve-out list |
| (uncited prose claim) `swarm_is_self_or_ancestor` reused for "who may stop a session" | CONFIRMED — independently traced to `comm_session.rs:1046` (`stop_allowed` computation in the session-stop handler), which does call `super::swarm_is_self_or_ancestor(&members, &req_session_id, &target_session)` exactly as claimed |
| (uncited prose claim) `SwarmAwaitCompleted` is the event `finalize_await` publishes | CONFIRMED — `comm_await.rs:196`: `Bus::global().publish(BusEvent::SwarmAwaitCompleted(SwarmAwaitCompleted { ... }))` inside `finalize_await` |
| (uncited prose claim) `BusEvent` "~30 variants" incl. `BatchProgress`/`FileTouch`/`SwarmOutputTail`/`SwarmAwaitCompleted` | CONFIRMED — enum has ~30 variants; all four named variants present at their approximate cited lines (399/401/403/409) |

## Narrative-level check

This chapter is a model example of the anti-hallucination discipline
working correctly: its own closing section, "What this chapter left open,"
explicitly and accurately carries forward every uncertainty the Phase 2
notes flagged rather than smoothing them into confident claims:

- Notes flag the soft-interrupt "safe points" injection claim as
  design-doc-sourced, not independently verified against
  `queue_soft_interrupt_for_session`'s implementation. The draft's prose
  (mid-chapter: "That specific injection-timing claim comes from the design
  doc, not something verified line by line here") and its closing section
  both correctly attribute this to the design doc rather than presenting it
  as a code-verified fact. Matches the notes exactly — no overreach.
- Notes flag `event_history: Arc<RwLock<VecDeque<SwarmEvent>>>` as cited
  only via the map/Chapter 4, not independently re-opened this pass. The
  draft's closing section says the same thing explicitly ("cited from
  Chapter 4's territory, not independently re-opened here"). No overreach.
- Notes flag the literal tool-call parameter shape (schema-level detail) as
  out of scope. The draft's closing section says the same. No overreach.
- The chapter does NOT import the map's flagged-but-unresolved
  "reparenting-on-departure" claim (`SWARM_ARCHITECTURE.md:40-45`) at all —
  it simply never mentions it, which is the safe choice.

No hallucinations found: nothing in this draft states as confident fact
anything the notes had flagged as open/unverified.

## TypeScript snippets

Both TS blocks are honestly labeled `// idiomatic TS equivalent — this
code does not exist in jcode`.

- `Bus` class (`extends EventEmitter`, `static global()`, `publish()`,
  `latestUpdateStatus()`) — correct Node `EventEmitter` API usage (`emit`,
  no error thrown when there are zero listeners, matching the real
  `broadcast::Sender::send`'s "ignored Result" behavior described in the
  surrounding prose). The prose explicitly and correctly caveats that
  `EventEmitter` doesn't drop events under backpressure the way a bounded
  `tokio::sync::broadcast` channel does — an honest, accurate distinction
  rather than papering over a behavioral mismatch between the two
  primitives.
- `statusFromString`/`recoverMemberStatus` — correct TS, and the logic
  faithfully mirrors the three real rewrite rules verified above (including
  the headless/non-terminal carve-out list).

No fixes needed beyond the one citation-range correction above.
