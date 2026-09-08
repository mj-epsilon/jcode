# Findings — Chapter 8 (Design-Pattern Glossary & Further Reading)

Reviewed against source directly on 2026-08-12, including cross-referencing
against notes-ch01.md through notes-ch04.md (chapters outside my primary
set) since this chapter claims to summarize the whole handbook. Spot-checked
roughly 20 of the table's ~30 rows against actual source, plus every
"Further reading" line-count/section-range claim. One major finding
(hallucination by omission) and one minor citation off-by-two; everything
else CONFIRMED, including several precise multi-hop citation chains.

## Major finding: hallucination by omission

### Row 1 of "Fan-out / fan-in combinators" drops a caveat that Chapters 1, 2, 3's own notes AND draft-ch03 itself treat as mandatory

Draft-ch08's table row:

> | Planner → fan-out → fan-in → integrate | `swarm.rs:1615-1701`, fan-in at
> `1671` | `try_join_all` — all-or-nothing, ordered `Vec<T>` |
> `Promise.all(tasks)` — near-exact match |

This presents the `try_join_all` planner→fan-out→fan-in pattern
(`run_swarm_message`/`run_swarm_task`) as an unqualified entry in the
cheat sheet, with no caveat. But:

- **`notes-ch01.md` §6** (open questions) explicitly flags: *"`run_swarm_task`/`run_swarm_message`... is **not** reachable from the live `swarm` tool at all... Chapter 1 (and especially Chapter 2/3) should not present `try_join_all` planner→fan-out→fan-in as 'what happens when an agent calls the swarm tool.'"*
- **`notes-ch02.md` §0** repeats this as a "critical scoping correction... stated up front," tracing both callers of `run_swarm_message` to debug-socket string commands (`swarm_message:`, `swarm_message_async:` in `debug_command_exec.rs`/`debug_jobs.rs`), not the `swarm` tool.
- **`notes-ch03.md`**'s very first section is titled "CROSS-CHECK CORRECTION... read this before drafting" and instructs: *"Phase 3 MUST label it accurately: a debug/dev-tooling code path... Say so explicitly in the chapter text."*
- **`draft-ch03.md` itself follows this instruction** — its Pattern 1 section opens with: *"One important caveat before we look at the code: this exact function, `run_swarm_message`, is reachable only from two debug-socket commands... it is **not** what happens when a production agent calls the `swarm` tool."*

I independently re-verified this is still true against current source:
`grep -rn "run_swarm_message\|run_swarm_task" crates/jcode-app-core/src/server/*.rs` shows the only external callers are `debug_command_exec.rs:138` and `debug_jobs.rs:95`; nothing in `communicate.rs` (the tool dispatch layer) or `comm_session.rs` (the live spawn path) calls either function.

**The problem**: Chapter 8 is explicitly framed as "a cheat sheet... use it
as an index back into the codebase" (its own opening line) and is exactly
the kind of page a reader might consult in isolation without having read
Chapter 3's careful caveat first. Presenting this row with zero
qualification — right alongside genuinely-live patterns like `spawn_swarm_agent`'s fire-and-forget task — risks exactly the misreading that Chapters 1, 2, and 3 all went out of their way to prevent: a reader could reasonably conclude this is what happens when a production agent's `swarm` tool call fans out work. It is not.

**Suggested fix**: append a short caveat to the row (or a footnote under the
table), e.g.: "debug/dev-tooling only — not reachable from the live `swarm` tool; see Chapter 3." This mirrors what `notes-ch08.md` item 5 itself should have carried forward from the earlier chapters' notes but didn't (that item also omits the caveat — this was dropped during Phase 2 analysis for Chapter 8, not introduced fresh during writing, but it's still wrong in the shipped chapter and should be fixed here).

## Minor finding

### MISCITED (off-by-two) — `swarm.rs:1233-1259` for `record_swarm_event`

Draft cites this range in the "Pub/sub & event delivery" table: "Replay
buffer beside a broadcast channel that has none | `event_history`,
`runtime.rs:107`, populated by `swarm.rs:1233-1259`". The function
signature is indeed at line 1233, but its closing brace is at **line
1257** — line 1258 is a blank line and line 1259 is actually the next
function's signature (`pub(super) async fn record_swarm_event_for_session(`).

**Correct citation: `swarm.rs:1233-1257`.**

`runtime.rs:107` for `event_history: Arc<RwLock<VecDeque<SwarmEvent>>>` is
CONFIRMED exact (independently re-verified — notes-ch07 had flagged this
field as relying on the map's citation, not independently re-opened in that
pass, so I checked it directly here rather than let it ride further
unverified).

## Everything else spot-checked: CONFIRMED

| Row / claim | Citation | Verdict |
| --- | --- | --- |
| `SwarmSpawnMode` enum (`Visible`/`Headless`/`Inline` default/`Auto`) | `jcode-config-types/src/lib.rs:646-657` | CONFIRMED — enum body spans exactly 646-657; `#[default]` on `Inline` confirmed |
| `DagError` + `Display`, "11 variants" | `dag/mod.rs:442-531` | CONFIRMED — enum declaration is 442-471, `Display` impl is 473-531, so 442-531 correctly spans "the enum + its Display impl" as the row's own label says; variant count independently recounted at exactly 11 (`UnknownNode`, `DuplicateNode`, `UnknownDependency`, `WouldCreateCycle`, `NotOwner`, `InvalidState`, `ThinArtifact`, `UnaddressedLowConfidence`, `UncoveredSiblings`, `StaleGateScope`, `GateMisuse`) |
| Layered delegation chain, `swarm.rs:1291-1527ish` | `update_member_status`→`_with_report`→`_with_report_tldr` | CONFIRMED — functions start at 1291/1319/1349 exactly; last function closes at line 1526, one line short of the stated "1527ish" — the explicit "ish" hedge makes this an honest approximation, not a false-precision claim |
| Tool schema `action` enum incl. `expand_node`/`complete_node`/`inject_gap` | `tool/communicate.rs:1959` | CONFIRMED — exact line |
| `"expand_node" =>` match arm | `communicate.rs:2629-2656` | CONFIRMED — match arm spans exactly 2629-2656 |
| `handle_comm_expand_node`, `dag::expand_node` call | `comm_graph.rs:339-402`, call at line 368 | CONFIRMED — function spans exactly 339-402; `dag::expand_node(...)` call is exactly at line 368 |
| `bridge.rs` doc comment + `to_task_graph`/`apply_task_graph` | `bridge.rs:1-9`, `94`, `124` | CONFIRMED — doc comment spans exactly lines 1-9; `to_task_graph` at exactly 94; `apply_task_graph` at exactly 124 |
| `NodeMeta` struct | `jcode-plan/src/lib.rs:117-146` | CONFIRMED — struct spans exactly 117-146 |
| `InterruptSignal` whole primitive | `jcode-agent-runtime/src/lib.rs:32-117` | CONFIRMED (as a whole-primitive pointer) — struct starts at 33 (32 is its `#[derive(Clone)]` line), `impl InterruptSignal` block (covering `fire`, `reset_if_epoch`, `notified`, `as_atomic`, `same_instance`) closes exactly at line 117 |
| `SwarmLifecycleStatus` "13 variants + Other" | `jcode-swarm-core/src/lib.rs:136-151` | CONFIRMED — enum spans exactly 136-151, 13 named variants + `Other` (re-confirmed, same as Chapter 7's independent count) |
| `docs/SWARM_ARCHITECTURE.md` (318 lines) | — | CONFIRMED — `wc -l` gives exactly 318 |
| `docs/SWARM_TASK_GRAPH.md` (605 lines) | — | CONFIRMED — `wc -l` gives exactly 605 |
| Further-reading section-range pointers (Roles 22-58, Lifecycle 89-106, Completion Report 108-124, Communication 194-254, §1a 54-98, §5 195-216, §6.4 275-304, §8a 407-475, §9 478-549) | various | All within 1-3 lines of the actual section boundaries (verified against `grep -n "^## "` headers in both docs) — these are prose "read around here" pointers, not literal code citations, and are close enough not to mislead a reader browsing to that page range |

## Narrative-level check against notes-ch01 through notes-ch04

- Confirmed the chapter does **not** present a nonexistent `enum Mode { AdHoc, Light, Deep }` — `notes-ch02.md` explicitly warns "there is no single Rust `enum` with exactly these three variant names" for the ad-hoc/light/deep *effort* axis. Chapter 8's table simply doesn't cover that axis at all (only `dag::Mode` Deep/Light rigor and `SwarmSpawnMode` rendering mode are in the table) — a safe omission, not a hallucination.
- Confirmed the chapter's explicit "don't conflate `dag::Mode` and `SwarmSpawnMode`" callout (opening paragraph and row 15's "**not `dag::Mode`**" label) matches the same warning independently raised in both `notes-ch02.md` and `notes-ch06.md`.
- Checked whether any row states something notes-ch01/02 flagged as an open question. `notes-ch02.md` open question #4 ("whether `dag::Mode` is literally consulted anywhere in the `comm_session.rs` mode-gate") is correctly *not* asserted anywhere in Chapter 8 — the chapter keeps the two mode concepts strictly separate without claiming a bridge exists between them.
- `notes-ch01.md`'s finding that "Worktree Manager" is design vocabulary with no corresponding Rust type is **not** contradicted anywhere in Chapter 8 — the chapter doesn't reference a worktree-manager role/type at all.
- The one real miss is the try_join_all caveat above — everything else cross-referenced against notes-ch01–04 held up.

## TypeScript / primitive-equivalent spot checks

Sampled several TS-equivalent column entries for correctness of claimed
Node/JS API behavior:

- "`Promise.all(tasks)` — near-exact match" for `try_join_all`: accurate
  characterization (both are all-or-nothing, both resolve to an ordered
  array) — matches `notes-ch03.md`'s own careful phrasing.
- "No built-in match for 'lossy pub/sub with an explicit lag signal'" for
  `broadcast::Receiver::recv()`/`Lagged(n)`: accurate — Node's
  `EventEmitter` has no bounded-ring-buffer/lag-signal concept built in.
- "`AbortController` + a manual 'generation' counter" for `InterruptSignal`:
  accurately flagged as "mostly moot" for single-threaded JS, matching
  `notes-ch04.md`'s explicit "Rust is meaningfully more complex here"
  framing rather than forcing a false 1:1 port.

No incorrect API claims found in the TS-equivalent column.
