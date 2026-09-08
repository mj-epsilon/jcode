# Findings — Chapter 3 (draft-ch03.md)

Reviewed adversarially against source at HEAD. Every inline citation
re-opened and diffed against the draft's quoted text and paraphrase;
every narrative claim about control flow / failure semantics
independently traced through the actual function bodies, not just the
citations shown.

## Citations checked

| # | Citation in draft | Verdict | Notes |
|---|---|---|---|
| 1 | `debug_command_exec.rs:132-139`, `debug_jobs.rs:77-95` (reachability caveat for `run_swarm_message`) | CONFIRMED | Re-verified independently of Chapters 1/2's identical claim — same two call sites, no third caller exists. |
| 2 | `swarm.rs:1657-1671` — `task_futures` build + `try_join_all` call | CONFIRMED | Exact match. Line 1657 is `let task_futures = tasks.iter().map(...)`, line 1671 is `let task_outputs = try_join_all(task_futures).await?;` — the range precisely brackets both, and the decision-table's split citation (`1671` for "the call", `1657-1670` for "fan-out build") is also exactly correct: line 1670 is the closing `});` of the map closure, line 1671 is the `try_join_all` call itself. |
| 3 | `tool/batch.rs:282-295` — `FuturesUnordered` construction | CONFIRMED | Exact match, character-for-character, including the trailing `.collect();` at line 295. |
| 4 | `tool/batch.rs:300-317` — drain loop | CONFIRMED | Line 300 is exactly `while let Some((i, tool_name, result)) = stream.next().await {`; the loop body (progress publish, `results.push`) matches the draft's paraphrase — no short-circuit on individual tool failure, matching the draft's/decision-table's "every result collected individually" claim. |
| 5 | `tool/batch.rs:319` — `results.sort_by_key(|(i, _, _)| *i)` | CONFIRMED | Exact line and exact code. |
| 6 | `MAX_PARALLEL = 10` "line 10 of the same file" | CONFIRMED | `const MAX_PARALLEL: usize = 10;` is exactly at line 10 of `batch.rs`. |
| 7 | Self-nested `batch` call rejection | CONFIRMED | `if Registry::resolve_tool_name(&tc.tool) == "batch" { return Err(...) }` present in `execute`, matches the draft's claim ("every self-nested `batch` call inside a `batch` call is rejected outright"). |
| 8 | `comm_await.rs:271-300` — the `tokio::select!` block | CONFIRMED | Exact match line-for-line, including both arms, the `Lagged`/`Closed` match arms, and the inline comment quoted verbatim ("Dropped events are recoverable..."). The draft's snippet omits the `crate::logging::info(...)` call inside the `Lagged` arm without an ellipsis marker — a minor abridgement, but it doesn't misrepresent the cited range (nothing quoted is fabricated, and the omitted line doesn't change the described behavior). |
| 9 | `comm_await.rs:209-303` — whole `spawn_or_resume_await_members` function | CONFIRMED | Function starts at line 209 (`pub(super) async fn spawn_or_resume_await_members(`) and its closing `}` (of the outer function, after the `tokio::spawn` block's own closing `});`) is exactly at line 303. |
| 10 | `finalize_await` publishing `BusEvent::SwarmAwaitCompleted` onto the *global* bus (Chapter 7 bridge, no explicit line cited in the draft body) | CONFIRMED | `comm_await.rs:196`, `Bus::global().publish(BusEvent::SwarmAwaitCompleted(...))`, inside `finalize_await` (declared at line 182). Matches the draft's narrative claim exactly, including the "different, process-wide channel from the per-swarm broadcast::Sender<SwarmEvent>" distinction. |

## Narrative/control-flow claims independently traced (not just cited, but walked through the actual function bodies)

- `run_swarm_task` (`swarm.rs:1528-1613`, referenced only narratively in the
  draft, not cited with an explicit file:line in the chapter body) — traced
  in full: confirms "forks a brand-new session and agent," "inherits the
  parent's exact model/provider/auth identity" (`session.provider_key =
  provider_key; session.route_api_method = route;`, with an inline comment
  matching the draft's paraphrase near-verbatim), and "strips
  `subagent`/`task`/`todo*` from the child's tool set" (`for blocked in
  ["subagent", "task", "todo", "todowrite", "todoread"] { allowed.remove(blocked); }`).
  All accurate.
- `run_swarm_message`'s three-step shape (plan → fan-out/fan-in →
  integrate), with steps 1 and 3 both running on the same locked
  coordinator `Arc<Mutex<Agent>>` — traced in full (`swarm.rs:1615-1701`)
  and confirmed exactly: `agent.lock().await` is taken separately for the
  planner call and again for the integration call, with the `try_join_all`
  fan-out in between using a cloned `Arc` per task.
- Pattern 2's "no short-circuit, every result collected individually"
  claim — confirmed: the drain loop unconditionally does
  `results.push((i, tool_name, result))` regardless of whether `result` is
  `Ok` or `Err`; failures are tracked in a `HashMap<usize, bool>`, not
  propagated as an early return.
- Pattern 3's core correctness claim ("the event itself is never trusted
  as the source of truth... every iteration re-reads status from shared
  state") — consistent with the full function body: `awaited_member_statuses(...)`
  is called at the top of the `loop`, before the `select!`, and the
  `Lagged`/normal-event arms both just `continue` back to the top of the
  loop rather than deriving the answer from the event payload.

## TypeScript code blocks

All three TS blocks (`runSwarmMessage`, `runBatchIncremental` generator,
`awaitMembers`) are honestly labeled `// idiomatic TS equivalent — this
code does not exist in jcode`.
- `Promise.all` fail-fast/non-cancelling semantics described in the prose
  are accurate (Promise.all does reject on first rejection; sibling
  promises are not cancelled, just ignored — correct as stated).
- The `runBatchIncremental` async generator's `Promise.race(pending.values())`
  approach is correctly caveated in the prose as O(n) per completion
  compared to a real resolver/counter pattern — an honest, not overstated,
  caveat.
- `awaitMembers`'s `Promise` wrapper with `setTimeout` + `EventEmitter`
  listener + explicit `isSatisfied()` re-check is a reasonable, technically
  sound sketch; correctly notes `EventEmitter` silently drops nothing
  (actually re-reads state on every event) analogous to the Rust
  `Lagged`-tolerant loop.
- No block claims to be real jcode code, and no fabricated Node/browser API
  usage found.

## Narrative-vs-Phase-2-notes divergence check

The draft's closing paragraph ("nobody traced whether an already-spawned
child session's on-disk state gets cleaned up when `try_join_all`
short-circuits...") directly and accurately preserves `notes-ch03.md`'s
explicitly flagged open question rather than asserting a settled fact
about cleanup behavior. This is correct anti-hallucination behavior — the
chapter states only the documented language-level fact (a dropped future
stops being polled) and explicitly declines to claim jcode does anything
further.

## Verdict

Zero MISCITED or UNVERIFIABLE findings. This is the cleanest chapter of the
three reviewed so far — every citation's line range precisely brackets the
quoted code, every narrative claim about control flow and failure
semantics holds up under independent tracing through the full function
bodies (not just the cited excerpts), and the chapter correctly preserves
rather than resolves the one open question flagged in Phase 2 notes.
