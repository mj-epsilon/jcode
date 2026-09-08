# Findings — Chapter 6 (The Task DAG / Deep Mode)

Reviewed against source directly on 2026-08-12. This chapter has three
citation-range findings (all MISCITED, all minor/boundary-level — none
invent behavior that doesn't exist) and is otherwise clean. Full citation
list below.

## Findings requiring correction

### 1. MISCITED — `dag/mod.rs:540` cites the wrong field entirely

Draft text (near the top, "Nothing else is allowed to touch the node list
directly" section):

```rust
// jcode: crates/jcode-plan/src/dag/mod.rs:540 — the node list is private,
// not `pub`. Every mutation has to go through the ops:: functions below.
nodes: Vec<TaskNode>
```

Actual source at that line range:
```
539: pub struct TaskGraph {
540:     pub mode: Mode,
541:     nodes: Vec<TaskNode>,
542: }
```

Line 540 is `pub mode: Mode,` — a *public* field, the opposite of the point
being illustrated. The private `nodes: Vec<TaskNode>` field the draft is
actually describing is on **line 541**, not 540. This is off by one line
and the line actually cited names the wrong thing. This same error also
appears in the Phase 2 notes (`notes-ch06.md`, section 1: "`dag/mod.rs:540`,
verified: `nodes: Vec<TaskNode>`"), so it was carried forward from analysis
into the draft rather than introduced fresh — but it's still wrong in the
shipped draft and should be fixed there.

**Correct citation: `crates/jcode-plan/src/dag/mod.rs:541`.**

### 2. MISCITED (minor, off-by-one) — `dag/mod.rs:72-101` excludes the impl block's own closing brace

Draft cites `dag/mod.rs:72-101` for the `NodeKind` enum + `impl NodeKind`
block (both `is_gate_kind` and `gate_kind`) shown together as one snippet.
Actual layout:
```
72:  pub enum NodeKind {
...
85:  }
87:  impl NodeKind {
...
101:     }        <- closes gate_kind()
102: }             <- closes impl NodeKind
```
The quoted snippet includes the `impl NodeKind { ... }` block in full
(both methods plus the block's own closing brace), but the cited range
stops at line 101 — one line short of the impl block's actual closing
brace on line 102. Not misleading about content (the code shown is
accurate), but the range doesn't fully cover what's quoted.

**Correct citation: `crates/jcode-plan/src/dag/mod.rs:72-102`.**

### 3. Imprecise citation — `dag/mod.rs:129-141 (enum), 141-204 (parse)` for `ConfidenceLevel`

The `ConfidenceLevel` enum itself (`Low`/`Medium`/`High`, the only code
actually shown in this snippet) spans lines 129-133, not 129-141. Lines
134-140 are the doc comment for the *separate* `parse` function and the
`impl ConfidenceLevel {` opening — not part of the enum. The two cited
ranges also double-count line 141 (labeled as the tail of "enum" and the
head of "parse" simultaneously). `parse`'s actual signature does start at
line 141 and the function does close at line 204, so the second half of
the citation is fine on its own.

**Correct citation: `crates/jcode-plan/src/dag/mod.rs:129-133 (enum),
141-204 (parse)`.**

## Everything else: CONFIRMED

| Citation | Claim | Verdict |
| --- | --- | --- |
| `jcode-swarm-core/src/lib.rs:390-433` | `append_deep_node_instructions` — only generates prompt text, doesn't implement the DAG | CONFIRMED — function signature at 390, closing brace at 433, exact match; confirmed it only builds a `String` via `push_str`, no graph mutation |
| `jcode-swarm-core/src/lib.rs:449-499` | `append_deep_gate_instructions` — same | CONFIRMED — signature at 449, closing brace at 499, exact match |
| `dag/mod.rs:37-49` | `Mode` enum (`Deep`/`Light`) + `requires_gates()` | CONFIRMED — exact match, including the doc-comment quote "One engine, two presets... rigor machinery (mandatory gates + strict artifact validation) is engaged" (verified verbatim against lines 33-35, draft's `...` correctly elides "(see doc section 1a)") |
| `dag/mod.rs:58-67` | `NodeOrigin`: `Seed \| Expand \| Gap \| Gate` | CONFIRMED — exact match; "a plan that never outgrew its seed is visibly under-explored" quote verified verbatim against the doc comment |
| `dag/mod.rs:107-116` | `NodeStatus`: `Queued \| Running \| Done \| Failed`, "Blocked" deliberately not stored | CONFIRMED — exact match; doc-comment quote "'Blocked' is intentionally not stored: it is computed from dependency state by the scheduler, so there is a single source of truth" verified verbatim |
| `dag/mod.rs:121-127` (quoted, uncited range but content checked) | `ConfidenceLevel` doc comment re: breadth signal | CONFIRMED verbatim word-for-word against source |
| `ops.rs:227-367` | `expand_node` full function | CONFIRMED — signature at 227, closing brace at 367 |
| `ops.rs:287` | `let mut staged = graph.clone();` | CONFIRMED — exact line and exact code |
| `ops.rs:361-364` | cycle check / `WouldCreateCycle` | CONFIRMED — exact match |
| `ops.rs:365` | `*graph = staged;` | CONFIRMED — exact line and exact code |
| `ops.rs:377-411` | `complete_node` full function | CONFIRMED — signature at 377, closing brace at 411 |
| `ops.rs:400` | `validate_artifact(...)` call | CONFIRMED — exact line |
| `ops.rs:402` | `validate_gate_pass(...)` call | CONFIRMED — exact line |
| `ops.rs:444-540` | `inject_from_gate` full function | CONFIRMED — signature at 444, closing brace at 540 |
| `ops.rs:505` | `NodeOrigin::Gap` for injected nodes, parented to gate's parent | CONFIRMED — exact line and exact code (`staged.push(spec_to_node(spec, parent.clone(), NodeOrigin::Gap));`) |
| `ops.rs:508-519` | gate re-queue block (`status = Queued`, `owner = None`, append new ids to `depends_on`) | CONFIRMED — exact block boundaries (508 opens the block, 519 closes it) |
| `ops.rs:593-628` | `mentions_node_id` word-boundary scanner | CONFIRMED — exact match, including the described `-_.:` id-character handling and the trailing `.`/`:` disambiguation logic |
| `ops.rs:789-878` | `validate_gate_pass`, three checks in order (stale scope, confidence debt, coverage debt) | CONFIRMED — exact function boundaries and exact check order; `GATE_COVERAGE_ENUMERATION_CAP = 20` confirmed at line 752; "declaring 'I did not check X' is the opposite of addressing X" quote verified verbatim (line 780) |
| `docs/SWARM_TASK_GRAPH.md` §9 (worked example, T0-T7) | scrollwm multimonitor scenario, facet names, gap topics, closing line | CONFIRMED — section exists at line 478 with exactly the T0-T7 subheadings described; gap topics "fullscreen on one output" / "mixed refresh rate" match verbatim; closing line "The graph is never drafted once; it grows wherever depth or gaps are found and shrinks in attention as subtrees collapse into synthesized artifacts" matches verbatim |

## Narrative-level check (cross-referenced against Phase 2 notes)

- The chapter's opening "correction" (jcode-swarm-core does NOT implement
  the DAG; the real engine is `crates/jcode-plan/src/dag/`) is faithfully
  carried from the notes' §0, itself independently verified — this is a
  case of the chapter *correctly* walking back an earlier wrong assumption,
  not introducing a new one. Good example of the anti-hallucination
  discipline working as intended.
- Notes §9 "Open questions" flags five things as explicitly unverified this
  pass: (1) exact wire JSON shape of `Request::CommExpandNode`/etc., (2)
  `schedule::ready_nodes`/`dispatch`/`assemble_input` bodies (not re-read
  this pass, only cited from Phase 1), (3) `bridge::to_task_graph`/
  `apply_task_graph` full bodies, (4) reparenting-on-departure, (5)
  `dag/sim.rs`. The draft cites **none** of these — it never mentions the
  wire protocol, `bridge.rs`, reparenting, or `sim.rs` at all. So no
  flagged-as-open-question material was smuggled in as confident fact.
- One claim in the draft *is* about `schedule.rs` behavior without a direct
  citation: "A node is blocked exactly when it's Queued and some dependency
  isn't Done yet." Notes flagged `schedule.rs` as not independently
  re-read this pass (relying on the Phase 1 map). I independently opened
  `crates/jcode-plan/src/dag/schedule.rs` myself to check this claim rather
  than let it ride on an unverified note: `ready_nodes` filters on
  `node.status == NodeStatus::Queued && deps_satisfied(graph, node)`, and
  `deps_satisfied` returns true only when every dependency `is_done()`
  (`schedule.rs:19-37`). The draft's claim is CONFIRMED accurate, so this
  is not a hallucination — but it's worth noting the draft got lucky here
  rather than the citation trail actually covering it (no `file:line` is
  given for this specific claim in the draft).

## TypeScript snippets

All TS blocks are honestly labeled `// idiomatic TS equivalent — this code
does not exist in jcode`. Checked for correctness:

- `requiresGates`, `NodeOrigin`/`NodeKind`/`NodeStatus` type aliases,
  `isGateKind`/`gateKindFor` — straightforward, correct, faithfully mirror
  the Rust match arms (`gateKindFor` correctly maps `implement`/`fix` to
  `verify` and everything else to `critique`, matching the real `_ =>
  NodeKind::Critique` catch-all).
- `isBlocked` — correct, matches the real (independently-verified, see
  above) scheduler semantics.
- `parseConfidence` — correct ordering (negation phrases before word-rung
  substrings), and every negation phrase used (`"not confident"`,
  `"unsure"`, `"uncertain"`, `"not sure"`) is a real phrase drawn from the
  actual Rust `NEGATIONS` array (`dag/mod.rs:149-157`, which has 7 entries;
  the TS sketch uses a real subset and says so — not an invention).
- `expandNode`/`hasCycle` — the cycle-detection DFS (white/gray/black
  coloring) is a textbook-correct algorithm; `expandNode`'s validation
  order (ownership → gate-misuse → state → empty-children → id
  collisions → unknown deps → stage → cycle-check → commit) faithfully
  mirrors the real function's order.
- `completeNode` — correct shape, matches the real function's
  validate-then-mutate structure.
- `injectFromGate` — correct, including parenting new nodes to `gate.parent`
  (matching the real `NodeOrigin::Gap` parenting rule) and re-queuing the
  gate with the new ids appended to `dependsOn`.
- `validateGatePass`/`mentionsNodeId` — correct three-check order (stale →
  confidence debt → coverage); the regex-based `mentionsNodeId` is a
  reasonable, clearly-simplified stand-in for the real hand-rolled scanner
  (doesn't claim to replicate the real trailing-`.`/`:` special case, which
  is fine since it's explicitly a teaching sketch, not a port). The
  character-class escape `/[.*+?^${}()|[\]\\]/g` is valid, standard JS
  regex-escaping syntax.

## One low-severity nit (not a citation error, no line was cited)

The draft's closing paragraph on `validate_gate_pass` says the error
"names the exact two things the gate could do next: call
`inject_from_gate` with follow-up nodes, or name the id in its own
findings." The actual error string (`DagError::UnaddressedLowConfidence`'s
`Display` impl, `dag/mod.rs:500-509`) says "inject_gap" (the tool-facing
action name), not "inject_from_gate" (the Rust function name) — the draft
never explains that these are the same underlying action at two different
layers (prompt-facing tool action vs. Rust function), so a careful reader
comparing this sentence against the actual error text could be confused by
the name mismatch. No citation was given for this specific sentence, so
it doesn't rise to MISCITED, but flagging it since a revision pass could
tighten it by either using "inject_gap" here or adding a one-line aside
noting the two names refer to the same action.
