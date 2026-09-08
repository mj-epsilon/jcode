# Findings — Chapter 4 (draft-ch04.md)

Reviewed adversarially against source at HEAD. Every inline citation
re-opened at its exact cited line range in both
`crates/jcode-app-core/src/server/runtime.rs` and
`crates/jcode-agent-runtime/src/lib.rs`, diffed character-for-character
against the draft's quoted code and prose paraphrases of doc comments.

## Citations checked

| # | Citation in draft | Verdict | Notes |
|---|---|---|---|
| 1 | `runtime.rs:27-37` — `RuntimeTaskScope` doc comment + struct | CONFIRMED | Exact match. Line 27 is the doc comment's first line, line 37 is the struct's closing `}` — the citation precisely brackets both the comment and the struct, more precise than the Phase 2 notes' own split citation (27-32 / 33-37). |
| 2 | `runtime.rs:40-59` — `RuntimeTaskScope::spawn` | CONFIRMED | Exact match, line-for-line, including the trailing closing brace at line 59. |
| 3 | `runtime.rs:61-74` — `RuntimeTaskScope::shutdown` | CONFIRMED | Exact match, including the full inline comment about the deadlock. |
| 4 | `runtime.rs:376-379` — `tokio::select!` race in `run_client_stream` | CONFIRMED | Exact match; confirmed this is inside `run_client_stream` (`fn` at line 333), matching the draft's framing. |
| 5 | `runtime.rs:156-185` — `spawn_main_accept_loop` | CONFIRMED | Function spans exactly 156-185 (`pub(super) fn spawn_main_accept_loop` at 156, closing `}` at 185); confirmed it uses bare `tokio::spawn` internally, matching the draft's claim. |
| 6 | `runtime.rs:187-219` — `spawn_debug_accept_loop` | CONFIRMED | Function spans exactly 187-219. |
| 7 | `runtime.rs:221-250` — `spawn_gateway_accept_loop` | CONFIRMED | Function spans exactly 221-250; confirmed it internally calls `self.tasks.spawn(...)`, matching the draft's specific claim that this accept loop (unlike the other two) is scope-registered. |
| 8 | `lib.rs:30-31` — `InterruptSignal` doc comment (quoted as blockquote) | CONFIRMED | Exact match, verbatim. |
| 9 | `lib.rs:32-40` — `InterruptSignal` struct | CONFIRMED | Exact match, including all three fields and `#[derive(Clone)]`. |
| 10 | `lib.rs:51-55` — `fire()` | CONFIRMED | Exact match. |
| 11 | `lib.rs:35-37` — `epoch` field doc comment (quoted as blockquote, including "(issue #428)") | CONFIRMED | Exact match, verbatim, including the real issue number — not fabricated. |
| 12 | `lib.rs:78-90` — `reset_if_epoch` | CONFIRMED | Exact match, line-for-line including the inline "A newer fire raced with the reset; restore it." comment. |
| 13 | `lib.rs:92-106` — `notified()` | CONFIRMED | Exact match. The draft's snippet omits the inline explanatory comment (lines 94-100) that's present in the real function body, but the draft's surrounding prose paraphrases that comment's content accurately elsewhere in the chapter — not a misrepresentation. |

## Narrative claims independently checked

- "2,000 times on a real multi-threaded runtime" test claim — CONFIRMED
  against `fire_never_loses_wakeup_while_notified_races`
  (`lib.rs:181-207`): uses `.worker_threads(2)` and `for i in 0..2000`.
- "`CancellationToken`... is one-shot — once cancelled, it stays
  cancelled; there's no 'reset'" — this is a claim about the external
  `tokio-util` crate's API, not jcode's own code, and checked against
  general `tokio_util::sync::CancellationToken` knowledge: correct, no
  `reset()`/un-cancel method exists on that type.
- "`AbortSignal` has no built-in `child_token()`" and the manual
  parent→child chaining workaround shown — correct as a statement about
  the real Web/Node `AbortController`/`AbortSignal` API.
- The two-spawn-styles claim (accept loops via bare `tokio::spawn` vs.
  per-connection handlers via `self.tasks.spawn`, with
  `spawn_gateway_accept_loop` as the one exception) — independently
  verified by reading all four function bodies directly, not just trusting
  the draft's summary. Accurate in every case.

## TypeScript code blocks

Four TS blocks (`RuntimeTaskScope` class, `shutdown` function,
`runConnection`, `InterruptSignal` class), all honestly labeled
`// idiomatic TS equivalent — this code does not exist in jcode`.
- `AbortController`/`AbortSignal` usage is correct throughout (`.abort()`,
  `.signal.aborted`, `addEventListener("abort", ..., { once: true })` — a
  real, correctly-used option).
- `EventTarget`/`Event`/`dispatchEvent` usage in the `InterruptSignal`
  sketch is correct standard Web/Node API usage (Node has had global
  `EventTarget`/`Event` since v15.4, so this isn't a fabricated primitive).
- `Promise.allSettled` used correctly for "wait for everything, tolerate
  individual failures" semantics at shutdown.
- The `shutdown` function's `scope["tasks"]` bracket-notation access to a
  private field is a minor stylistic wart (bypassing TS's `private` via
  string indexing) but is clearly presented as an illustrative sketch, not
  production code, and doesn't misrepresent any real API — not flagged as
  a substantive issue.

## Narrative-vs-Phase-2-notes divergence check

The draft's "An honest gap, not filled in" paragraph — stating that the
actual call sites where Esc/Ctrl+C invokes `.fire()`, or where the agent
turn loop calls `.notified()`/`.is_set()`, were not traced — directly and
accurately preserves the open question flagged in `notes-ch04.md`'s
closing section. The draft does not upgrade this into a confident claim
about where the wiring happens; it correctly limits itself to "this is
what the primitive does and why," matching the notes' own stated boundary.

## Verdict

Zero MISCITED or UNVERIFIABLE findings. Every one of the 13 inline
citations checked resolves exactly — this chapter's citation precision
is the best of the four (notably, its own citations are in some cases
*more* precise than the Phase 2 notes they were drawn from, e.g. the
combined `runtime.rs:27-37` doc-comment-plus-struct citation). All four
TS blocks are honestly labeled and use real, correctly-applied Node/Web
APIs. The chapter's one open question is correctly preserved as open
rather than resolved into an unverified claim.
