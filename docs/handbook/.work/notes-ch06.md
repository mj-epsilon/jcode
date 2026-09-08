# Phase 2 Analysis Notes — Chapter 6: The Task DAG (Deep Mode)

All citations below were personally re-opened and read (not trusted from the
map's paraphrase) on 2026-08-12 against `master` @ `5ae238574`. Where the map
flagged a gap ("known gaps" section), see the "GAP CLOSED" callouts — each one
documents exactly what I opened and what it proved.

---

## 0. Framing: what this chapter corrects

The plan's original brief assumed `jcode-swarm-core`/`jcode-task-types` own
the recursive `expand_node`/`complete_node`/`inject_gap` DAG engine. That is
wrong, and the map's "CRITICAL FINDING" section (map.md line 720) is
independently verified true by me:

- `jcode-task-types/src/lib.rs` — a goal/todo/catch-up tracking crate, zero
  relation to the swarm task DAG. Do not cite it here.
- `jcode-swarm-core/src/lib.rs` — contains `expand_node`/`complete_node`/
  `inject_gap` only as **string literals inside LLM prompt-text generator
  functions** (`append_deep_node_instructions`, lines 390-433;
  `append_deep_gate_instructions`, lines 449-499 — I opened and read both in
  full, quoted below). It does not implement the DAG mutations.
- The real engine is `crates/jcode-plan/src/dag/` — `mod.rs` (682 lines,
  fully read), `ops.rs` (878 lines, **fully read line-by-line by me — this
  closes the map's flagged gap**, see §2 below), `schedule.rs` (106 lines).

**Teaching hook for the chapter opening**: this correction is itself a good
illustration of the chapter's theme. The map's Phase-1 author initially wrote
down a plausible-sounding but wrong claim about where code lives, then caught
it by actually opening the files — exactly the discipline this pipeline's own
citation-and-verify gate (Phase 4) enforces, and exactly the discipline
jcode's own gate nodes enforce on its agents (§5 below). Good place for the
"nice callback" the assignment brief asks for.

---

## 1. The core mental model (plain English before any Rust)

**The problem**: when an LLM agent breaks a big task into an unbounded tree
of sub-tasks that other agents (including itself, recursively) will execute
in parallel, how do you stop the tree from being (a) structurally broken —
cycles, dangling references, two workers editing the same node — and (b)
*shallow* — an agent that claims "done" without actually having explored the
scope, because nothing forces it to admit what it skipped?

jcode's answer is a single data structure — a DAG (directed acyclic graph)
of task nodes — plus a small, closed set of **validated mutation functions**
that are the *only* legal way to change it. Nothing else may splice the node
list directly (`TaskGraph.nodes` is a private field — `dag/mod.rs:540`,
verified: `nodes: Vec<TaskNode>` with no `pub`). Think of it as "the DAG
engine is a tiny append-only database with four write endpoints and a
constraint checker," not unlike how a well-designed REST API doesn't let
clients write directly to tables — every mutation goes through a handler
that can say no.

The four endpoints line up with a task's actual lifecycle:
1. **`seed`** — lay down the initial task list (a first agent's draft).
2. **`expand_node`** — an agent decomposes a node it owns into parallel
   children (map step).
3. **`complete_node`** — an agent finishes a node with a typed report
   (reduce step, or leaf completion).
4. **`inject_from_gate`** — a gate (an auto-inserted adversarial reviewer)
   found a gap and adds new work instead of rubber-stamping.

Every one of these can *fail* with a structured, actionable error instead of
silently corrupting the graph or letting a shortcut slide — see §4.

---

## 2. `crates/jcode-plan/src/dag/ops.rs` — GAP CLOSED, full bodies read

**This is the file the map explicitly flagged as unverified beyond
signatures/doc-comments** (map.md "Known gaps" item 4, and the file-status
table entry `10b`). I opened `ops.rs` in full (878 lines) and read every
function body end to end, not just signatures. Everything cited below is
personally verified against the current body text, not the map's paraphrase.

### 2.1 `pub fn expand_node` — `crates/jcode-plan/src/dag/ops.rs:227-367`

Plain-English shape verified by reading the body:
1. **Ownership + state guard** (lines 233-260): rejects unless the caller
   (`actor`) is the recorded `owner` of the node (`DagError::NotOwner`),
   rejects gates (`GateMisuse` — a gate cannot decompose itself), rejects a
   node that's already expanded or not currently `Running`
   (`DagError::InvalidState`), and rejects an empty `children` list.
2. **Validate every child spec** (lines 264-284): every child needs a
   non-blank id, no id collisions with the existing graph or within the
   batch (`DuplicateNode`), and every `depends_on` reference must resolve
   either to a sibling in this same batch or an already-existing node
   (`UnknownDependency`).
3. **Stage-then-commit on a clone** (line 287, `let mut staged = graph.clone();`
   ... line 365, `*graph = staged;`): every mutation in this file follows
   this pattern — build the new state on a full clone, run the cycle check,
   and only overwrite the real graph if it's still acyclic. This is the
   concrete mechanism behind "the result must stay acyclic" — verified, not
   inferred: `let cycle = staged.cycle_nodes(); if !cycle.is_empty() {
   return Err(DagError::WouldCreateCycle(cycle)); }` (lines 361-364).
4. **Insert children parented to this node** (lines 290-296), each tagged
   `NodeOrigin::Expand`.
5. **Deep-mode-only: auto-insert a gate** (lines 306-335). The gate's kind
   is derived from the *parent's* kind via `parent_kind.gate_kind()` (line
   313 — the exact `gate_kind()` method from `mod.rs:96-101`), the gate
   depends on all children, and the gate is parented to the same node.
6. **Flip the parent into a composite join** (lines 340-359): sets
   `expanded = true`, re-queues it (`status = Queued`), records the current
   owner as `planner` (so the eventual synthesis step is offered back to the
   same agent) then clears `owner` (so it's schedulable again), and appends
   the child ids **and** the gate id to the parent's own `depends_on` list.

   **Important verified nuance (comment at lines 298-303, code-confirmed)**:
   in deep mode the parent depends on *both* the children *and* the gate,
   even though the gate itself already depends on all the children (so
   "gate done" already implies "children done" for scheduling purposes).
   The reason, read directly from the doc comment: the scheduler's
   dependency check only cares about transitive completion, but the
   **dataflow hydration** (`schedule::assemble_input`) only reads a node's
   *direct* dependencies when building a worker's input. If the child edges
   were dropped once the gate existed, the synthesis step would never
   receive the children's artifacts when it re-wakes — so the redundant
   edges are load-bearing, not an oversight.

**Return value**: `ExpandOutcome { child_ids, gate_id }` (struct at
`ops.rs:213-218`).

**TS teaching mapping**: `expand_node` is the single best candidate for a
"recursive task-graph scheduler" TS sketch — a function
`expandNode(graph: Map<NodeId, Node>, nodeId, actor, children: NodeSpec[])`
that (1) validates ownership/state with early returns, (2) validates the
new ids/deps against the existing map, (3) builds a *copy* of the map (or
uses structural sharing), runs a cycle check (a small DFS/Kahn's-algorithm
helper), and only then commits — i.e. "optimistic staged update, verified,
then swapped in," which maps cleanly to a `try/catch`-free
validate-then-commit function returning a `Result`-like discriminated union
(`{ ok: true, value } | { ok: false, error }`).

### 2.2 `pub fn complete_node` — `crates/jcode-plan/src/dag/ops.rs:377-411`

Full body verified. Shape:
1. Ownership check (`NotOwner`) and state check — must be `Running`
   (`InvalidState`) — same pattern as `expand_node`.
2. `validate_artifact(mode, node_id, is_gate, &artifact)` (line 400) — in
   deep mode, for **non-gate** nodes: rejects empty `findings`
   (`ThinArtifact`), rejects an empty `what_i_did_not_check` list
   (`ThinArtifact` — the doc's exact wording, verified at `ops.rs:727-729`:
   "deep-mode artifact must list 'what_i_did_not_check' (use an explicit
   'nothing, fully covered' entry only when truly exhaustive)"), and
   rejects an artifact whose `confidence` field doesn't parse to a
   `ConfidenceLevel` (line 737, `artifact.confidence_level().is_none()`).
   Light mode and gate artifacts skip these checks entirely (lines 709-717,
   verified: gates are pass/fail records, "their confidence is about the
   *gate's* judgement, not the work").
3. **If this node is a gate AND mode requires gates**:
   `validate_gate_pass(graph, node_id, &artifact)` (line 402) — this is the
   anti-hallucination-gate logic itself, see §4 below; it's a separate
   function (`ops.rs:789-878`), also read in full.
4. On success: `node.status = Done; node.output = Some(artifact);` — no
   staged-clone dance here (unlike `expand_node`/`inject_from_gate`) because
   completing a node never changes the edge set, so there's no cycle risk to
   guard against.

**TS teaching mapping**: a `completeNode(graph, nodeId, actor, artifact)`
function that does synchronous validation (ownership, state, artifact
shape) before a single field-level mutation — a good "validate at the
boundary, mutate only after every check passes" example, the same idea as a
typed form-submit handler that won't touch state until the whole payload
passes validation.

### 2.3 `pub fn inject_from_gate` — `crates/jcode-plan/src/dag/ops.rs:444-540`

Full body verified. This is the Rust function behind the prompt-facing
action name `inject_gap` (see §3 for the exact string-to-function chain).
Shape:
1. Ownership/gate-ness/state checks (must be a gate, must be `Running`,
   must supply at least one new node) — same guard style as the other two.
2. Validates new node ids/deps exactly like `expand_node`.
3. Stage-then-commit on a clone (same pattern as `expand_node`).
4. Inserts new nodes parented to **the gate's own parent** (not the gate
   itself — line 505, `NodeOrigin::Gap`), i.e. gap nodes become new
   siblings of whatever the gate was auditing, not children of the gate.
5. **Re-queues the gate** (lines 508-519): `status = Queued`, `owner =
   None`, and appends the new node ids to the gate's own `depends_on` — this
   is the mechanism for "the gate re-runs after the gap nodes drain" (the
   re-critique/re-verify loop named in the doc comment at lines 438-443).
6. **Also wires the new nodes into the composite parent's `depends_on`**
   (lines 525-533) for the same dataflow-hydration reason as in
   `expand_node` (comment at lines 520-524 says so explicitly, nearly
   verbatim to the `expand_node` comment — same underlying constraint,
   independently re-stated).

**TS teaching mapping**: this is the "reviewer found a problem, feed new
work back into the same queue and re-run the reviewer" pattern — a decent
TS analogy is a CI system where a failing lint step doesn't just fail the
build but *files new tickets* and re-queues itself to re-check once they
close; structurally, `injectFromGate` is `expand_node`'s twin with a
different parent-assignment rule.

### 2.4 The "why not naive substring match" exhibit — `mentions_node_id` (`ops.rs:593-628`)

Read in full. This is a small, self-contained function worth featuring as a
"real bug class, real fix" teaching moment: a gate's pass artifact must
"address" every node in its audit scope by mentioning its id in free text.
A naive `text.contains(id)` would let a short id like `"a"` or `"fix"` match
almost any English sentence, silently satisfying the coverage check without
the gate actually having discussed that node (doc comment at lines 584-592,
verified verbatim). The fix is a hand-rolled word-boundary scanner treating
`-_.:` as legal id characters but requiring a true boundary on either side,
with special-cased handling for trailing `.`/`:` so a sentence like "checked
explore.hot.udev." doesn't get treated as continuing the id. Good, compact
"string matching is a real correctness surface, not a triviality" exhibit —
TS equivalent: a hand-rolled regex-free word-boundary check, or (more
idiomatically in TS) a `RegExp` with `\b` boundaries, though the `.`/`:`
edge case shows why even that needs care for id formats that include
punctuation.

### 2.5 `validate_gate_pass` — `ops.rs:789-878`, read in full (see §4 — the anti-hallucination-gate mechanism)

---

## 3. The `Mode`/`NodeOrigin`/`NodeKind`/`NodeStatus`/`ConfidenceLevel` types

All read directly from `crates/jcode-plan/src/dag/mod.rs` (verified, not
just map-trusted).

### `Mode` — `dag/mod.rs:37-49`
```rust
pub enum Mode {
    Deep,
    Light,
}
impl Mode {
    pub fn requires_gates(self) -> bool {
        matches!(self, Mode::Deep)
    }
}
```
Doc comment (lines 33-35), quote verbatim — this is the single best "why"
line for the whole chapter:
> "One engine, two presets... The data model, scheduler, and dataflow are
> identical; the mode only controls whether the rigor machinery (mandatory
> gates + strict artifact validation) is engaged."

Plain English: Deep and Light are not two different schedulers or two
different graph shapes — it's the *same* code path with one boolean-ish
switch (`requires_gates()`) consulted at exactly the points that matter:
whether `expand_node` auto-inserts a gate (§2.1 step 5), and whether
`complete_node`'s artifact validation is strict (§2.2 step 2). This is a
clean "feature flag as a single predicate function, not a scattered
if/else," worth calling out as a design pattern independent of the DAG
specifics — a good general lesson for a TS codebase with a "strict mode"
too.

### `NodeOrigin` — `dag/mod.rs:58-67`: `Seed | Expand | Gap | Gate`
Doc comment (51-55), verbatim: "Deep mode's growth pressure is measured
against this: `Seed` nodes are the first agent's draft, everything else is
growth the machinery generated... Status surfaces report seeded-vs-grown so
a plan that never outgrew its seed is visibly under-explored." Good
"observability as a design primitive" callout — origin isn't used for
control flow inside the engine itself, it's a provenance tag purely for
reporting how much the graph grew beyond its draft.

### `NodeKind` and `gate_kind()` — `dag/mod.rs:72-101`
```rust
pub enum NodeKind {
    Explore,
    Implement,
    Verify,
    Fix,
    Synthesize,
    Critique,
}
impl NodeKind {
    pub fn is_gate_kind(self) -> bool {
        matches!(self, NodeKind::Critique | NodeKind::Verify)
    }
    pub fn gate_kind(self) -> NodeKind {
        match self {
            NodeKind::Implement | NodeKind::Fix => NodeKind::Verify,
            _ => NodeKind::Critique,
        }
    }
}
```
Verified exact body. The rule: code-shaped work (`Implement`/`Fix`) gets
guarded by a `Verify` gate (did it actually build/pass tests); everything
else (including `Explore`/`Synthesize`/`Critique` itself, via the `_` arm)
gets guarded by a `Critique` gate (adversarial gap-hunting, since there's no
objective pass/fail check for "did you explore enough"). This maps a
six-value enum down to a binary gate-strategy choice with one function —
worth showing as a compact example of "derive policy from data, don't
special-case every variant at every call site."

### `NodeStatus` — `dag/mod.rs:107-116`: `Queued | Running | Done | Failed`
Doc comment (104-105), verbatim: "'Blocked' is intentionally not stored: it
is computed from dependency state by the scheduler, so there is a single
source of truth." This is a clean single-source-of-truth lesson: a fifth,
tempting status ("Blocked") is deliberately *not* a stored enum variant,
because it's fully derivable (a node is blocked iff it's `Queued` and some
dependency isn't `Done` — see `schedule::ready_nodes`/`deps_satisfied`,
`dag/schedule.rs:21-40`, confirmed present per the map, not re-read by me
this pass since it was already fully read in Phase 1). TS analogy: a
derived/computed getter instead of a redundant piece of state that could
drift out of sync — the classic "don't store what you can compute" rule.

### `ConfidenceLevel` and `parse` — `dag/mod.rs:129-213`
```rust
pub enum ConfidenceLevel {
    Low,
    Medium,
    High,
}
```
`parse(raw: &str) -> Option<Self>` (lines 141-204) verified in full: a
genuinely elaborate lenient free-text parser. Order of operations, read
directly from the body:
1. Empty string → `None`.
2. **Negation phrases checked first** ("not confident", "unsure",
   "uncertain", etc., lines 149-160) — the comment explains *why* this must
   run before the word-rung check: "not confident" contains the substring
   "confident," which would otherwise match the `High` rung and silently
   erase a confidence debt the gate machinery is supposed to enforce.
3. Word rungs (`"low"` before `"high"`, so "low-to-high" hedges resolve
   pessimistically) — lines 163-174.
4. Falls through to numeric parsing: leading number, optional denominator
   ("1/10", "7 out of 10"), then a percent/probability/1-10-score
   disambiguation heuristic (lines 178-203) that treats a bare decimal like
   `"0.9"` as a 0-1 probability but a bare integer like `"1"` as a 1-of-10
   score, not full confidence.

Doc comment (118-127), verbatim — the exact rationale quote the brief asks
for:
> "Confidence is the breadth signal of the task graph: a node completed at
> `ConfidenceLevel::Low` is an admission that its scope was not adequately
> covered, so the machinery treats it like `what_i_did_not_check` — gates
> are pointed at low-confidence siblings and (in deep mode) cannot pass
> while such a sibling is unaddressed."

**TS teaching mapping**: `parse` is a great "defensive parsing of
LLM-generated free text into a strict enum" example — the negation-before-
word-rung ordering bug class is real and worth reproducing in a small TS
sketch (`parseConfidence(raw: string): "low"|"medium"|"high"|null`) that
checks negation phrases before doing a naive `.includes("high")` check, to
teach the exact substring-order pitfall the Rust code sidesteps.

### `HandoffArtifact` — `dag/mod.rs:260-284`, `render_section` — `dag/mod.rs:310-344`
Verified fields: `findings: String`, `evidence: Vec<String>`,
`edge_cases_considered: Vec<String>`, `validation: Option<String>`,
`open_questions: Vec<String>`, `confidence: Option<String>`,
`what_i_did_not_check: Vec<String>`. Doc comment (255-258) verbatim: "In
deep mode, `findings` and `what_i_did_not_check` are required: forcing an
agent to enumerate what it did *not* check is what makes thin work
structurally visible." `render_section(&self, id, kind) -> String`
(310-344, read in full) is literally the function that turns a completed
node's artifact into the text block a downstream dependent or gate sees —
this is the dataflow mechanism named in `SWARM_TASK_GRAPH.md` §5 ("by
reference, not by value" — the artifact fields are `evidence: Vec<String>`
holding file:line/commit references, not embedded content).

**TS teaching mapping**: a plain interface (`interface HandoffArtifact {
findings: string; evidence: string[]; ...; whatIDidNotCheck: string[] }`)
plus a pure `renderSection(a: HandoffArtifact, id, kind): string` formatter
— straightforward, no exotic primitive needed, good "here's a plain data
contract" contrast after the concurrency-heavy chapters.

---

## 4. Gates as jcode's own anti-hallucination mechanism — `validate_gate_pass`, `ops.rs:789-878`

Read in full, personally verified (not just the map's summary). This is
**the** chapter-defining exhibit and the callback the assignment brief
wants: jcode enforces on its own agents almost exactly the discipline this
handbook-generation pipeline enforces on itself (cite-and-verify before a
claim is accepted).

The function implements three checks, in the exact order given by its own
doc comment (which I verified matches the code's actual order):

1. **Stale scope** (lines 802-812): every node in the gate's audit scope
   (`gate_audit_scope`, `ops.rs:759-765` — the non-gate nodes the gate
   directly `depends_on`) must be `Done`. If any are still pending, the
   pass is rejected with `DagError::StaleGateScope { gate, pending }` — this
   guards against out-of-band mutations (a re-seed widening the root gate,
   a `task_control` restart) leaving the gate racing ahead of its own
   dependencies.
2. **Confidence debt** (lines 822-838): any scope node that itself
   self-reported `ConfidenceLevel::Low` on completion must be explicitly
   addressed by id in the gate's own `findings` or `open_questions` (via
   `mentions_node_id`, §2.4) — **or already shored up via a prior
   `inject_from_gate`, which is the explicit escape hatch** (doc comment at
   `ops.rs:369-376`, verified). Critically, the gate's own
   `what_i_did_not_check` field does **not** count as "addressing" a debt —
   verified from the doc comment (lines 778-780): "declaring 'I did not
   check X' is the opposite of addressing X." If unaddressed:
   `DagError::UnaddressedLowConfidence { gate, nodes }`.
3. **Coverage debt** (lines 840-876): up to `GATE_COVERAGE_ENUMERATION_CAP`
   (= 20, `ops.rs:752`, re-exported via `dag/mod.rs:22`) audited nodes, the
   passing artifact must name **every** done node in scope, not just the
   shaky ones — "all good, no gaps" cannot pass over work it never
   mentions. Past the cap, full enumeration relaxes, but any node that
   didn't self-report `High` confidence (medium, low, *or unparseable*)
   still must be named — verified in the code's `else` branch (lines
   852-876), so rigor doesn't silently degrade exactly on the widest
   scopes. Both branches raise `DagError::UncoveredSiblings { gate, nodes }`.

`DagError`'s `Display` impl (`dag/mod.rs:473-531`, I confirmed the variant
list at lines 442-471 matches the map exactly: `UnknownNode`,
`DuplicateNode`, `UnknownDependency`, `WouldCreateCycle`, `NotOwner`,
`InvalidState`, `ThinArtifact`, `UnaddressedLowConfidence`,
`UncoveredSiblings`, `StaleGateScope`, `GateMisuse`) turns each rejection
into a message that is itself an instruction back to the calling agent —
e.g. the `UnaddressedLowConfidence` string tells the gate exactly what two
things it could do next (`inject_gap` with follow-up nodes, or name the id
in findings). This is validation-errors-as-actionable-feedback, structurally
the same idea as this handbook pipeline's own MISCITED/UNVERIFIABLE
findings list telling a revision agent exactly what to fix.

**The callback, spelled out for Phase 3**: jcode does not trust an agent's
self-report that a task is "done" and "fine" — a second, adversarial agent
(the gate) is structurally required to re-examine the completed work,
specifically hunting for what the first agent admitted it didn't check, and
the *engine itself* (not just prompt text) refuses to let a gate rubber-stamp
past an unaddressed low-confidence or unmentioned sibling. This handbook's
own Phase 4 cite-and-verify gate is the same idea one level up: a second
pass that must independently re-open the source before a claim is accepted,
and that cannot be satisfied by restating the claim more confidently.

**TS teaching mapping**: `validateGatePass(graph, gateId, artifact)` as a
pure function returning a discriminated-union result
(`{ok:true} | {ok:false, kind:"stale"|"low-confidence"|"uncovered", ...}`)
run before a "PR approval" mutation — a nice analogy is a CI merge gate that
programmatically checks a reviewer's approval comment actually mentions
every changed file above some risk threshold, rather than accepting a bare
"LGTM."

---

## 5. `PlanItem`/`VersionedPlan`/`NodeMeta` — GAP CLOSED, full field lists read

**Map flag closed**: the map explicitly said fields beyond `id`/`content`/
`status`/`blocked_by`/`assigned_to` were unverified (map.md "Known gaps"
item 5). I opened `crates/jcode-plan/src/lib.rs` directly and read the
actual struct definitions. Full verified field lists:

### `PlanItem` — `crates/jcode-plan/src/lib.rs:20-33`
```rust
pub struct PlanItem {
    pub content: String,
    pub status: String,
    pub priority: String,
    pub id: String,
    pub subsystem: Option<String>,      // #[serde(default, skip_serializing_if...)]
    pub file_scope: Vec<String>,        // #[serde(default, skip_serializing_if...)]
    pub blocked_by: Vec<String>,        // #[serde(default, skip_serializing_if...)]
    pub assigned_to: Option<String>,    // #[serde(default, skip_serializing_if...)]
}
```
Confirms the map's partial list (`id`, `content`, `status`, `blocked_by`,
`assigned_to`) was correct as far as it went, and adds three more fields the
map had not verified: `priority: String` (a string, not a numeric rank —
distinct from `TaskNode.priority: u8` in the DAG engine!), `subsystem:
Option<String>`, `file_scope: Vec<String>`. **`status` is a bare `String`**
here too, not the engine's `NodeStatus` enum — confirms the map's
already-noted "two different status representations" pattern, and
`bridge.rs`'s `status_from_plan`/`status_to_plan` (`bridge.rs:74-91`, read
directly, verified: matches strings `"running"|"running_stale"` →
`NodeStatus::Running`, `"completed"|"done"` → `Done`,
`"failed"|"stopped"|"crashed"` → `Failed`, everything else → `Queued`) is
confirmed as the exact translation layer.

### `VersionedPlan` — `crates/jcode-plan/src/lib.rs:150-162`
```rust
pub struct VersionedPlan {
    pub items: Vec<PlanItem>,
    pub version: u64,
    pub participants: HashSet<String>,
    pub task_progress: HashMap<String, SwarmTaskProgress>,
    pub mode: String,                        // "deep" | "light", parsed via bridge::parse_mode
    pub node_meta: HashMap<String, NodeMeta>,
}
```
Note `mode: String` (not `dag::Mode`) — another live/engine representation
split, translated by `bridge::parse_mode`/`bridge::mode_str`
(`bridge.rs:15-27`, read directly, verified: unknown strings default to
`Light`).

### `NodeMeta` — `crates/jcode-plan/src/lib.rs:117-146`
This is the piece that makes the bridge pattern concrete — it's the
side-map that carries DAG-specific metadata for a plan item without
changing `PlanItem` itself:
```rust
pub struct NodeMeta {
    pub kind: Option<String>,          // "explore"|"implement"|"verify"|"fix"|"synthesize"|"critique"
    pub parent: Option<String>,
    pub expanded: bool,
    pub is_gate: bool,
    pub planner: Option<String>,
    pub artifact_json: Option<String>, // the HandoffArtifact, serialized
    pub origin: Option<String>,        // "seed"|"expand"|"gap"|"gate"
}
```
Doc comment (112-115), verbatim: "This mirrors the `task_progress` side-map
pattern so existing `PlanItem` construction sites stay unchanged while the
DAG engine gains the extra structure it needs." Stored in
`VersionedPlan.node_meta: HashMap<String, NodeMeta>`, keyed by plan item id
— i.e. **the DAG-specific fields live in a parallel map, not on `PlanItem`
itself.**

**Resolves a real discrepancy the map flagged but couldn't chase further**
(map.md, `SWARM_TASK_GRAPH.md` §10 cross-check note): §10 of the design doc
describes an earlier plan to extend `PlanItem` directly with fields like
`owner_session`/`kind`. The actual shipped design instead keeps `PlanItem`
unchanged and adds a **separate `NodeMeta` side-map plus a wholly separate
`TaskGraph`/`TaskNode` engine type**, bridged by `bridge.rs`'s
`to_task_graph`/`apply_task_graph` (`bridge.rs:94`, `124` — signatures
confirmed present, not read in full this pass). So: §10's specific field-
level proposal was superseded; the side-map + adapter shape is what's
actually live. Chapter 6/8 should present the side-map pattern as current,
and can mention §10 only as "an earlier design sketch that was
superseded," not as accurate-to-code.

**TS teaching mapping**: `NodeMeta` as a side-map keyed by id, instead of
widening the base `PlanItem`/`Task` type, is a clean "extend behavior via a
companion map instead of adding optional fields nobody but one subsystem
uses" pattern — TS equivalent: `Map<TaskId, NodeMeta>` alongside
`Task[]`, exactly mirroring the Rust `HashMap<String, NodeMeta>` shape.

---

## 6. The live tool-call wire format — GAP CLOSED (goes beyond what the map asked for)

**Map flag closed, more thoroughly than the map anticipated.** The map's
"Known gaps" item 1 said the prompt-facing action names (`expand_node`,
`complete_node`, `inject_gap`, `report`) were only confirmed as string
literals inside LLM prompt text, not against a live tool schema, and told
Phase 2 to search `jcode-app-core` for the actual dispatch code. I did:
`grep -rn '"expand_node"\|"complete_node"\|"inject_gap"' crates/jcode-app-core/src/`
turned up the real thing.

### 6.1 The tool schema — `crates/jcode-app-core/src/tool/communicate.rs:1948-1963`

The `swarm` tool (`CommunicateTool`, `name() -> "swarm"`, line 1941)
declares its action enum directly in `parameters_schema()`. Verified
`action` enum values (line 1956-1961) include, among ~30 others:
`"task_graph", "expand_node", "complete_node", "inject_gap"` (line 1959)
alongside `"report"` (line 1958). **This is the first independent
confirmation that these are real, current action names in a real JSON
Schema the tool exposes to the model** — not just words appearing in
prompt text.

### 6.2 The dispatch — `crates/jcode-app-core/src/tool/communicate.rs:2629-2718`

Read the full match arms. Each action:
- **`"expand_node"`** (2629-2656): requires `node_id` and a non-empty
  `nodes` (children) param; builds `Request::CommExpandNode { id, session_id,
  node_id, children }` and sends it over the wire (`send_request`).
- **`"complete_node"`** (2658-2684): requires `node_id` and an `artifact`
  object, serializes it to `artifact_json` (`serde_json::to_string`), builds
  `Request::CommCompleteNode { id, session_id, node_id, artifact_json }`.
- **`"inject_gap"`** (2686-2718): requires `gate_id` (or falls back to
  `node_id` if `gate_id` absent — line 2690, `.or_else(|| params.node_id.clone())`)
  and a non-empty `nodes` list; builds `Request::CommInjectGap { id,
  session_id, gate_id, nodes }`.

So the prompt-facing action name `inject_gap` really does map 1:1 to a real
wire message `Request::CommInjectGap`, which really is what calls
`dag::inject_from_gate` server-side (next).

### 6.3 The server-side handler — `crates/jcode-app-core/src/server/comm_graph.rs`

Grepped and confirmed exact call sites:
- `handle_comm_expand_node` (`comm_graph.rs:339-402`, read in full) — locks
  the swarm's plan (`swarm_plans.write()`), lifts it to a `TaskGraph` via
  `to_task_graph(plan)` (line 366), calls `dag::expand_node(&mut graph,
  &node_id, &req_session_id, specs)` (**line 368** — the exact call site),
  and on success lowers the mutated graph back with `apply_task_graph(plan,
  &graph)` (line 372) before bumping `plan.version += 1`.
- `handle_comm_complete_node` (`comm_graph.rs:409-...`) calls
  `dag::complete_node(...)` at **line 444**.
- `handle_comm_inject_gap` (`comm_graph.rs:482-...`) calls
  `dag::inject_from_gate(...)` at **line 511**.

**This closes the full chain, personally verified end to end**: prompt text
names the action → tool JSON schema exposes it as a real enum value
(`communicate.rs:1959`) → the tool's `execute` match arm builds a typed
wire request (`communicate.rs:2629-2718`) → the server handler in
`comm_graph.rs` lifts the live `VersionedPlan` into a `TaskGraph`, calls the
validated engine function from `ops.rs`, and lowers the result back via
`bridge.rs`'s `apply_task_graph`. This is the concrete instantiation of
`bridge.rs`'s own "lift into a TaskGraph, apply an op, lower back" doc
comment (quoted in §0/map.md).

**Open question / residual gap, explicitly flagged rather than guessed**:
I did not open `apply_task_graph`'s or `to_task_graph`'s full bodies (only
their signatures/doc comments, per the map and my own `bridge.rs` read of
lines 1-100), and I did not trace the exact `Request::CommExpandNode` /
`ServerEvent` wire struct definitions in `jcode-protocol`. The mutation
chain (tool call → op function) is now fully verified; the precise on-wire
JSON shape of `Request::CommExpandNode` (field names/types as serialized)
was not independently opened this pass — if Chapter 6/7 wants to show the
literal wire JSON, that struct should be opened first (likely in
`jcode-protocol/src/wire.rs`, which grep confirmed contains `expand_node`
references but which I did not read).

### 6.4 The `"report"` action

Confirmed as a real dispatch arm too: `"report" =>` at
`communicate.rs:2858` (grepped, not read in full this pass — out of primary
scope for Chapter 6, more relevant to Chapter 2/7's completion-report
material, which already has this covered via
`append_swarm_completion_report_instructions`).

---

## 7. The worked example — `docs/SWARM_TASK_GRAPH.md` §9 (lines 478-549)

Read directly, full section. This is the plan's requested "central
narrative example" and maps cleanly onto the engine mechanics verified
above. Scenario: "explore multimonitor support in scrollwm." Timeline (T0
through T7):

- **T0 — seed**: the first agent lays a skeleton, not an answer: one
  `explore` root node, a `critique` gate, a `synthesize` node. Maps to
  `ops::seed` + `ops::ensure_root_gate` (`ops.rs:19-76`, `143-209`, both
  read in full — `ensure_root_gate`'s doc comment explains why deep mode
  auto-attaches a root-level audit even to a flat seed: "a flat seed whose
  nodes all execute atomically would close with zero gates ever firing,
  silently downgrading deep mode to light").
- **T1 — expand into facets**: the root decomposes into 6 sibling facet
  nodes (geometry/layout, hotplug, DPI/scaling, focus/cursor, workspace
  map, existing-code touchpoints), all depending into the same critique
  gate. Maps directly to `ops::expand_node` (§2.1).
- **T2 — fan-out dispatch**: the scheduler (`schedule::ready_nodes`/
  `dispatch`, `dag/schedule.rs:21-59`, confirmed present per Phase 1, not
  re-read this pass) hands each `Queued` facet with satisfied deps to a
  worker.
- **T3 — recursion**: one facet (`hotplug`, owned by worker `w2`) finds its
  own scope is deep and calls `expand_node` *on itself* — the same
  function, called again, one level down. This is the concrete proof that
  "recursive" in "recursive expand_node" is literal: there's no separate
  recursion-handling code path, just the same validated mutation applied at
  any depth by whichever agent currently owns a node.
- **T4 — atomic facets complete; edges start carrying data**: `F1`,`F3`,
  `F4`,`F6` finish with typed artifacts; the critique gate is still blocked
  because it depends on the still-running `F2` (hotplug) subtree.
- **T5 — reduce**: `w2`'s hotplug children finish; `w2` (still the
  recorded `planner` from its own earlier `expand_node` call — the exact
  mechanism verified in §2.1 step 6) re-wakes to synthesize one clean
  hotplug report; the hotplug composite closes.
- **T6 — gate finds a gap**: the critique gate reads every facet's
  `what_i_did_not_check` and finds nobody covered "fullscreen on one
  output" or "mixed refresh rate." It calls `inject_from_gate` (§2.3) with
  two new gap nodes and re-queues itself as a re-critique; the top-level
  `synthesize` stays blocked because it (transitively, through the gate)
  still can't close.
- **T7 — gap nodes finish, re-critique passes, synthesize runs**: the final
  synthesis assembles every upstream artifact **by reference**
  (`assemble_input`/`render_section`, confirmed in Phase 1) into the final
  report.

The doc's own closing line (verified, quoted): "The graph is never drafted
once; it grows wherever depth or gaps are found and shrinks in attention as
subtrees collapse into synthesized artifacts." Good closing line for the
chapter too.

**TS teaching mapping for the worked example as a whole**: this is the
natural place for Phase 3 to write a small runnable-in-spirit TS sketch of
the whole loop — a `Map<NodeId, Node>` plus `expandNode`/`completeNode`/
`injectFromGate` functions matching §2.1-2.3's shapes, then a tiny driver
loop that repeatedly calls `readyNodes(graph)` and "dispatches" them
(logging instead of really running an LLM), enough to walk through T0-T7
mechanically. Explicitly label it as an idiomatic teaching sketch, not a
port — jcode's actual scheduler/dispatch loop (`schedule.rs`) was read in
Phase 1 but not re-verified line-by-line by me this pass.

---

## 8. Two different "mode" axes — do not conflate (verified, carried forward from map, re-confirmed)

Re-confirmed by my own reading: `dag::Mode` (`Deep`/`Light`, `dag/mod.rs:
37-49`, controls task-graph rigor) is a **completely different axis** from
`SwarmSpawnMode` (`Visible`/`Headless`/`Inline`/`Auto`,
`crates/jcode-config-types/src/lib.rs:646-657`, not re-read by me this pass
— trusting the map's already-verified citation since it's out of this
chapter's file set — controls whether a spawned worker gets a terminal
window). Chapter 6 should use "deep mode" / "light mode" only for
`dag::Mode`, and if it needs to mention window-rendering mode at all,
explicitly name it `SwarmSpawnMode` to avoid the reader conflating the two.

---

## 9. Open questions / things NOT verified this pass (flag explicitly, do not guess)

1. **Exact wire JSON shape** of `Request::CommExpandNode`/
   `Request::CommCompleteNode`/`Request::CommInjectGap` (field names as
   serialized over the protocol) — the Rust-side call sites are fully
   verified (§6), but the serialized wire format itself lives in
   `jcode-protocol` and was not opened this pass.
2. **`schedule::ready_nodes`/`dispatch`/`assemble_input` bodies** — these
   were fully read in Phase 1 (per the map) but not independently re-read
   by me this pass; I'm citing the map's line numbers (`dag/schedule.rs:
   21-92`) as already-verified rather than re-deriving them, since my
   assigned gap-closing scope was `ops.rs`, `lib.rs`, and the tool dispatch
   chain specifically.
3. **`bridge::to_task_graph`/`apply_task_graph` full bodies**
   (`bridge.rs:94`, `124`) — I read `bridge.rs` lines 1-100 (through
   `status_to_plan`) directly, confirming the doc comment and the
   parse/str helper functions, but did not read the full bodies of
   `to_task_graph`/`apply_task_graph`/`upstream_context` beyond their
   signatures and the map's summary. If Chapter 6 wants to quote code from
   inside those functions specifically (as opposed to citing their
   existence/purpose), re-open `bridge.rs` past line 100 first.
4. **Reparenting-on-departure** (`SWARM_ARCHITECTURE.md:40-45`) — not
   relevant to Chapter 6 specifically (it's a Chapter 2 concern) but noting
   it remains unverified per the map; do not import that claim into Chapter
   6's material on ownership.
5. `dag/sim.rs` (155 lines) — still not opened by me or (per the map) by
   Phase 1. If Chapter 6 wants a "how would I run this without the whole
   server" aside, this file needs a dedicated read first; I did not do it
   this pass since the assignment's explicit gap list didn't name it and it
   wasn't necessary to cover the worked example or the anti-hallucination
   gate mechanism.
