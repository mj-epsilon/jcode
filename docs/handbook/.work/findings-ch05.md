# Findings — Chapter 5 (Shared State Without Locks-as-a-Crutch)

Reviewed against source directly (not against the Phase 2 notes' own
verification claims) on 2026-08-12.

## Zero findings. Every citation confirmed exact.

All citations checked out against source, byte-for-byte for code blocks and
faithfully-elided for the quoted doc comments. Full list below for the
record.

| # | Citation in draft | Claim | Verdict |
| --- | --- | --- | --- |
| 1 | `docs/SWARM_ARCHITECTURE.md:307-311` | "optimistic by default (no locks)... Coordination happens via DM or channel" quote | CONFIRMED — line 307 is the `## Conflict Handling (No Locks)` header, 309 and 311 are the exact two bullet lines quoted (ellipsis correctly marks the elided line 310) |
| 2 | `crates/jcode-app-core/src/server/state.rs:106-113` | `SwarmState` struct, 4 `Arc<RwLock<HashMap<...>>>` fields | CONFIRMED — exact match, including doc comment and `#[derive(Clone)]` |
| 3 | `crates/jcode-app-core/src/server/state.rs:15-31` | `BACKGROUND_TOOL_SIGNALS` doc comment + `static` declaration | CONFIRMED — exact match; range correctly ends at line 31 (the two-line static decl), doesn't bleed into the accessor fns that start at 34 |
| 4 | `crates/jcode-base/src/bus.rs:499-502` | `Bus::global()` `OnceLock` singleton | CONFIRMED — exact match |
| 5 | `swarm.rs:108-113` (second mention, prose only) | "debounce bookkeeping... shows up again" pointing at `pending_swarm_status_broadcasts()` | CONFIRMED — matches the function at those exact lines |
| 6 | `crates/jcode-app-core/src/server/swarm.rs:94-100` | quoted doc comment re: ~240 KB payload measurement | CONFIRMED — exact match (quote correctly ellipsizes lines 95 and 98-99, keeps 94/96/97/100 verbatim) |
| 7 | `swarm.rs:87-88, 102-113` | debounce constants + `PendingSwarmStatusBroadcast` struct + `pending_swarm_status_broadcasts()` fn | CONFIRMED — exact match on all three pieces at their stated line numbers |
| 8 | `swarm.rs:758-781` (trimmed from `742-812`) | `broadcast_swarm_status`'s threshold-check + `should_spawn` decision block | CONFIRMED — both the trim range and the stated full-function range (742-812, ends exactly at the closing `}` on line 812) are exact |
| 9 | `client_state.rs:25-53` (called at line 897) | `should_debounce_attach_model_prefetch` — leading-edge rate limit | CONFIRMED — range is exact (25 = const, 53 = closing brace); call site at line 897 confirmed exact (`if should_debounce_attach_model_prefetch(&provider_name) {`) |
| 10 | `crates/jcode-base/src/bus.rs:542-606` (prose mention only, no snippet) | `Bus::publish_models_updated` — same debounce family, different bookkeeping | CONFIRMED — function starts at 542 (`pub fn publish_models_updated(&self) {`), closing brace at 606, matches exactly |

## Narrative-level check

- The "two senses of no locks" framing matches the Phase 2 notes' resolution
  exactly and does not overstate anything the notes flagged as uncertain.
  Notes explicitly say "No open/unresolved questions for this chapter's core
  material" — draft doesn't introduce any new confident claims beyond that.
- Notes flagged `swarm.rs:2174-2189` (a test-only doc comment about ordering
  subtlety in the immediate/non-debounced path) as citable only as a vague
  aside, never as production behavior. The draft does not cite or mention
  this test-only item at all — it simply omitted it, which is the safe
  choice and produces no hallucination risk.
- The "at least three times, independently" debounce-recurrence claim in
  the draft's Part B intro is directly supported by the notes' own
  independent `grep -rniE debounce` search (occurrence 3 was a genuine
  addition the Phase 1 map didn't have, confirmed by direct read in the
  notes and re-confirmed by me above at `client_state.rs:25-53`).

## TypeScript snippets

All four TS blocks are honestly labeled `// idiomatic TS equivalent — this
code does not exist in jcode`. Spot-checked each for API correctness:

- `SwarmState` TS class — plain `Map`/`Set` fields, no issues.
- `backgroundToolSignals` Map + two accessor functions — correct `Map.get`/
  `Map.set` usage, matches the Rust accessors' shape.
- `getBus()` lazy singleton — standard, correct module-level lazy-init
  pattern.
- `broadcastSwarmStatus` debounce — correct use of `setTimeout` (named
  function expression `function flush()` recursing via its own name
  binding is valid JS), `ReturnType<typeof setTimeout>` is the portable way
  to type a timer handle across Node/browser. Logic mirrors the Rust
  scheduled/dirty flow faithfully.
- `shouldDebounceAttachModelPrefetch` — correct `Date.now()`/`Map` usage,
  matches the Rust leading-edge-rate-limit shape.

No fixes needed for this chapter.
