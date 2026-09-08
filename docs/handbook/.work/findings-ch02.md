# Findings — Chapter 2 (draft-ch02.md)

Reviewed adversarially against source at HEAD. Every inline citation was
re-opened at its cited line range and diffed against the draft's quoted
text; every narrative claim about function signatures/line counts was
independently recomputed rather than trusted.

## Citations checked

| # | Citation in draft | Verdict | Notes |
|---|---|---|---|
| 1 | `debug_command_exec.rs:132-139` | CONFIRMED | Exact match. |
| 2 | `debug_jobs.rs:77-95` | CONFIRMED | Exact match. |
| 3 | `tool/communicate.rs:1939-1946` — `impl Tool for CommunicateTool { fn name(&self) -> &str { "swarm" }` | CONFIRMED | Exact match. |
| 4 | `tool/communicate.rs:2009-2021, 2043-2047` — spawn field schemas | **MISCITED (partial)** | The snippet shown also quotes the `"initial_message"` field (`"Alias of prompt for spawn; wins when both are set."`), but that field is actually at lines **2022-2025**, which falls in the gap between the two cited ranges (2009-2021 and 2043-2047) and is not covered by either. The quoted text is accurate, just not fully covered by the given citation. **Correct citation**: extend the first range to `2009-2025` (or cite `2022-2025` separately for `initial_message`). |
| 5 | `tool/communicate.rs:2154-2187` — `anyOf` schema for `label` requirement | CONFIRMED | Exact match, including both inline comments (the general multi-action rationale and the Gemini issue #655 comment). |
| 6 | `tool/communicate.rs:2720-2751` — `"spawn" =>` dispatch | CONFIRMED | Exact match. |
| 7 | `comm_session.rs:557-579` — `spawn_swarm_agent` signature | CONFIRMED | Exact match. However the draft's inline comment `// ... 10 more shared-state/channel params` is **inaccurate**: the full signature has 21 parameters; the draft explicitly lists 9 (`req_session_id` through `sessions`), so the true remainder is **12 more**, not 10. Minor but it is a specific, checkable claim about API shape. |
| 8 | `comm_session.rs:591` — `resolved_spawn_mode` fallback | CONFIRMED | Exact match, line 591 is exactly `let resolved_spawn_mode = spawn_mode.unwrap_or(agents_config.swarm_spawn_mode);`. |
| 9 | `comm_session.rs:622-652` — visible/headless branch | CONFIRMED | Exact match. |
| 10 | `comm_session.rs:654-695` — headless fallback | CONFIRMED | Exact match, including confirming `create_headless_session`'s call-site argument order (`mcp_pool`, then `Some(req_session_id.to_string())`, then `HeadlessMemoryScope::RealProject`) lines up positionally with the signature cited later. |
| 11 | `comm_session.rs:772-822` — fire-and-forget `tokio::spawn` block | CONFIRMED | Exact match, line-for-line. |
| 12 | `jcode-config-types/src/lib.rs:643-657` — `SwarmSpawnMode` enum | CONFIRMED | Exact match, including all doc comments and the `#[default]` on `Inline`. |
| 13 | `jcode-base/src/prompt.rs:129-131` — `is_deep_swarm_effort` | CONFIRMED | Exact match. |
| 14 | `session_effort.rs:1-19` — module doc comment | CONFIRMED | Exact match (the quoted deadlock rationale is verbatim). |
| 15 | `comm_session.rs:1234-1431` — `ensure_spawn_coordinator_swarm` | CONFIRMED (trivial overshoot) | Function's actual closing `}` is at line **1430**; line 1431 is a blank line before the next item (`#[cfg(test)]` at 1432), not another function's code. Not misleading, but the technically precise range is `1234-1430`. Not flagging as a real MISCITED — noting only because the draft's own math (`1431-1234+1=198`) baked in the off-by-one. |
| 16 | `comm_session.rs:1330-1348` — root-effort mode gate | CONFIRMED | Exact match, including the doc comment. |
| 17 | `comm_session.rs:1350-1361` — hard cap check | CONFIRMED | Exact match. |
| 18 | `comm_session.rs:1363-1378` — configurable budget check | CONFIRMED | Exact match, including the historical-bug comment. |
| 19 | `swarm.rs:26-32` — `MAX_SWARM_MEMBERS` re-export doc comment | CONFIRMED | Exact match. |
| 20 | `swarm.rs:234-236` — `member_consumes_swarm_capacity` | CONFIRMED | Exact match. |
| 21 | `comm_session.rs:502-517` "(relevant field only)" — `SwarmMember` construction, ancestry section | **MISCITED (partial)** | The snippet as shown includes `role: "agent".to_string(),` — but that field is at line **519**, two lines past the cited range's end (517). The `report_back_to_session_id` line shown *is* correctly inside the range (line 517). **Correct citation**: extend to `502-519` to cover the `role` line actually quoted, or drop the `role` line from the snippet if keeping the tighter range. |
| 22 | `headless.rs:37-54` — `create_headless_session` signature | CONFIRMED | Exact match, including parameter order and the trailing `report_back_to_session_id: Option<String>, memory_scope: HeadlessMemoryScope,`. |
| 23 | `swarm.rs:995-1219` — `remove_session_from_swarm`, "the largest function in the file" | **MISCITED** | The function's actual closing `}` is at line **1213**; lines 1214-1219 are a blank line followed by the *doc comment and signature line* of an unrelated next function, `set_member_task_label` (`pub(super) async fn set_member_task_label(` begins at 1219). The cited range bleeds 6 lines into a different function. The draft's own "225 lines" figure is computed from the wrong endpoint (`1219-995+1=225`); the real function is 219 lines (`1213-995+1`). **Correct citation**: `swarm.rs:995-1213`. |
| 24 | `swarm.rs:1115-1121` — reparenting rationale doc comment | CONFIRMED | Exact match. |
| 25 | `swarm.rs:1122-1142` — `fallback_parent` computation | CONFIRMED | Exact match. |
| 26 | `swarm.rs:1143-1156` — reparenting loop | CONFIRMED | Exact match. |
| 27 | `swarm.rs:1054-1069` — coordinator election on departure | **MISCITED** | The `let new_coordinator = { ... };` block the draft quotes (opening through the closing `};`) actually spans lines **1056-1070** in source — the cited range starts 2 lines early (1054-1055 are `let mut elected_coordinator = None;` / `if was_coordinator {`, not shown in the snippet) and ends 1 line short, excluding the block's own closing `};` at line 1070, which *is* shown in the draft's snippet. **Correct citation**: `swarm.rs:1056-1070`. |

## Narrative claims spot-checked

- "The full action list runs to nearly forty values" — the actual enum at
  `communicate.rs:1954-1963` has **43** entries, which is more accurately
  "more than forty" than "nearly forty." Cosmetic wording issue only, not
  worth a MISCITED classification, but flagged in case a revision pass
  wants tighter phrasing.
- The core scoping correction — `run_swarm_task`/`run_swarm_message` is
  debug-only, not the live spawn path — is restated and re-verified in this
  chapter exactly as in Chapter 1's version. Re-confirmed independently
  (same two call sites, same absence of any third caller).
- `spawn_swarm_agent` = 271 lines (`557-827`) — CONFIRMED exactly; line 827
  is the function's actual closing `}` right after `Ok(new_session_id)`.

## TypeScript code blocks

All seven TS blocks (`SpawnRequest`/`handleSpawnAction`, `spawnWorker`,
`attemptSpawn`, `canSpawnRecursively`, `canSpawnAnother`, `fireFirstTurn`,
`registerMember`, `removeMember`) are honestly labeled `// idiomatic TS
equivalent — this code does not exist in jcode`. Spot-checked for API
correctness:
- No fabricated Node/browser APIs used anywhere (plain `Map`, `Promise`
  chaining, `.then()/.catch()`, `Array.prototype.sort()`/`.min`-via-sort
  patterns).
- `removeMember`'s `.sort()[0]` for "lowest session id" relies on default
  lexicographic string sort, which is correct behavior for string ids (no
  numeric-sort bug).
- `fireFirstTurn`'s prose correctly describes `no-floating-promises` as a
  real, common ESLint rule name — accurate.
- No block claims to be real jcode code; all are clearly contrasted against
  the preceding Rust.

## Narrative-vs-Phase-2-notes divergence check

Compared against `notes-ch02.md` §8 "Open questions for later phases."
Both open items are correctly preserved as open in the draft's closing
section ("What this chapter didn't cover"): (1) the exact `headless.rs`
line where `SwarmMember` gets constructed for the headless path is
explicitly flagged as not traced, matching notes item 1; (2) the
relationship between the effort-string gate and a separate `dag::Mode`
enum is explicitly flagged as unconfirmed, matching notes item 4. No
instance found where the draft upgrades a notes-flagged open question into
a stated fact.

## Verdict

Four MISCITED findings, all minor (line-range boundary errors of 2-6 lines
that don't change the underlying claim's truth, plus one inaccurate "10
more params" count). No UNVERIFIABLE findings — everything cited actually
exists and says approximately what the draft claims. A revision pass should
tighten these four citations:
1. `communicate.rs:2009-2021, 2043-2047` → extend to include `2022-2025`
   (`initial_message` field).
2. `comm_session.rs:502-517` → extend to `502-519` (covers the quoted
   `role` field).
3. `swarm.rs:995-1219` → correct to `995-1213`.
4. `swarm.rs:1054-1069` → correct to `1056-1070`.
Plus one narrative fix: "10 more shared-state/channel params" → "12 more".
