# Chapter 6 — The Task DAG (Deep Mode)

## A correction, made in public

Before we get to the code, a small confession about how this very chapter
got written.

The first pass at mapping this codebase assumed the recursive task-graph
engine — the thing that lets an agent break a task into sub-tasks, and those
sub-tasks into further sub-tasks — lived in a crate called `jcode-swarm-core`.
That assumption was wrong. `jcode-swarm-core` does contain the strings
`expand_node`, `complete_node`, and `inject_gap` — but only inside
functions that generate prompt *text* for the LLM
(`append_deep_node_instructions`, `append_deep_gate_instructions`,
`crates/jcode-swarm-core/src/lib.rs:390-433` and `449-499`). It doesn't
implement the graph at all.

The real engine lives in `crates/jcode-plan/src/dag/` — `mod.rs`, `ops.rs`,
`schedule.rs`. The mistake was caught only because a later pass opened the
files directly instead of trusting the earlier summary, and re-derived every
claim from the actual function bodies.

Hold onto that story. By the end of this chapter you'll see that jcode bakes
the exact same discipline — don't trust a self-report, re-check the actual
evidence — into its own task graph, as a first-class engine feature rather
than a one-off writing habit.

## The problem, in plain English

Say you let an LLM agent break a big task into an unbounded tree of
sub-tasks, and those sub-tasks spawn further sub-tasks, and other agents
(including itself) pick them up and work on them in parallel. Two things can
go wrong:

1. **The tree gets structurally broken.** Cycles. Sub-tasks that reference
   work that doesn't exist. Two workers editing the same node at once.
2. **The tree gets shallow.** An agent says "done" without having actually
   explored the scope it was given — and nothing forces it to admit what it
   skipped.

jcode's answer is one data structure — a DAG (directed acyclic graph) of
task nodes — plus a small, closed set of **validated mutation functions**
that are the only legal way to change it. Nothing else is allowed to touch
the node list directly:

```rust
// jcode: crates/jcode-plan/src/dag/mod.rs:541 — the node list is private,
// not `pub`. Every mutation has to go through the ops:: functions below.
nodes: Vec<TaskNode>
```

Think of it like a REST API that never lets a client write straight to a
database table — every write goes through a handler that can say no. The
four "endpoints" here line up with a task's actual lifecycle:

1. **seed** — lay down the initial task list.
2. **`expand_node`** — an agent decomposes a node it owns into children.
3. **`complete_node`** — an agent finishes a node with a typed report.
4. **`inject_from_gate`** — a reviewer node finds a gap and adds work
   instead of rubber-stamping.

Every one of these can *fail* with a structured, actionable error instead of
silently corrupting the graph. We'll get to what that looks like.

## One engine, two presets: `Mode`

```rust
// jcode: crates/jcode-plan/src/dag/mod.rs:37-49
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

The doc comment above this enum is worth quoting verbatim, because it's the
single best "why" sentence in the whole engine:

> "One engine, two presets... The data model, scheduler, and dataflow are
> identical; the mode only controls whether the rigor machinery (mandatory
> gates + strict artifact validation) is engaged."

Deep and Light aren't two different schedulers, or two different graph
shapes. It's the *same* code, gated behind one predicate function
(`requires_gates()`), consulted at exactly the two points that matter: does
`expand_node` auto-insert a reviewer, and does `complete_node` enforce
strict validation on the finished work. That's a clean pattern worth
stealing on its own, independent of task graphs: a feature flag reduced to
one function, checked at its call sites, instead of `if mode == "deep"`
scattered through the codebase.

```ts
// idiomatic TS equivalent — this code does not exist in jcode
type Mode = "deep" | "light";

function requiresGates(mode: Mode): boolean {
  return mode === "deep";
}
```

One note before we go further: this "mode" has nothing to do with whether a
spawned worker gets its own terminal window (that's a separate setting,
`SwarmSpawnMode`, covered in Chapter 2 and Chapter 8's glossary). Two
different axes, same word — don't conflate them.

## Where a node came from: `NodeOrigin`

```rust
// jcode: crates/jcode-plan/src/dag/mod.rs:58-67
// Seed | Expand | Gap | Gate
```

`Seed` nodes are the first agent's rough draft. Everything else —
`Expand`, `Gap`, `Gate` — is growth the machinery generated afterward. The
doc comment explains the point of tagging this at all: "a plan that never
outgrew its seed is visibly under-explored." `NodeOrigin` doesn't drive any
control flow inside the engine — it's pure provenance, there so a status
report can say "6 of 9 nodes in this graph were never part of the original
draft," which is a useful signal that real decomposition happened instead
of an agent skimming the surface and calling it done.

```ts
// idiomatic TS equivalent — this code does not exist in jcode
type NodeOrigin = "seed" | "expand" | "gap" | "gate";
```

## What kind of node, and who reviews it: `NodeKind` and `gate_kind()`

```rust
// jcode: crates/jcode-plan/src/dag/mod.rs:72-102
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

`gate_kind()` is the rule that decides who reviews a piece of work. Code-shaped
work — `Implement`, `Fix` — gets a `Verify` gate: something with an
objective pass/fail signal, like "did it build, did the tests pass."
Everything else — `Explore`, `Synthesize`, even `Critique` reviewing
itself — falls through to a `Critique` gate: an adversarial second read
hunting for gaps, because "did you explore enough" has no compiler to check
it. One function collapses a six-value enum down to a binary
review-strategy choice, instead of special-casing each variant at every
call site that needs to know how to review it.

```ts
// idiomatic TS equivalent — this code does not exist in jcode
type NodeKind = "explore" | "implement" | "verify" | "fix" | "synthesize" | "critique";

function isGateKind(kind: NodeKind): boolean {
  return kind === "critique" || kind === "verify";
}

function gateKindFor(kind: NodeKind): NodeKind {
  if (kind === "implement" || kind === "fix") return "verify";
  return "critique";
}
```

## Four states, not five: `NodeStatus`

```rust
// jcode: crates/jcode-plan/src/dag/mod.rs:107-116
// Queued | Running | Done | Failed
```

Notice what's missing: `Blocked`. The doc comment says why, and it's a
lesson worth generalizing: "'Blocked' is intentionally not stored: it is
computed from dependency state by the scheduler, so there is a single
source of truth." A node is blocked exactly when it's `Queued` and some
dependency isn't `Done` yet — that's fully derivable from data already on
hand, so storing it as a fifth enum variant would just create a second copy
of the truth that could quietly drift out of sync with the real one.

```ts
// idiomatic TS equivalent — this code does not exist in jcode
type NodeStatus = "queued" | "running" | "done" | "failed";

// "blocked" is never stored — it's a computed view:
function isBlocked(node: Node, graph: Map<string, Node>): boolean {
  return (
    node.status === "queued" &&
    node.dependsOn.some((id) => graph.get(id)?.status !== "done")
  );
}
```

Don't store what you can compute — the same rule that keeps a derived
`fullName` getter off a database row.

## Reading confidence out of free text: `ConfidenceLevel::parse`

When a node finishes, the agent that did the work self-reports how
confident it is. That confidence has to come out of free-form LLM text, not
a clean radio button, which makes parsing it a small adversarial problem in
its own right.

```rust
// jcode: crates/jcode-plan/src/dag/mod.rs:129-133 (enum), 141-204 (parse)
pub enum ConfidenceLevel {
    Low,
    Medium,
    High,
}
```

The doc comment states the stakes plainly:

> "Confidence is the breadth signal of the task graph: a node completed at
> `ConfidenceLevel::Low` is an admission that its scope was not adequately
> covered, so the machinery treats it like `what_i_did_not_check` — gates
> are pointed at low-confidence siblings and (in deep mode) cannot pass
> while such a sibling is unaddressed."

`parse` is genuinely careful about a real bug class. It checks **negation
phrases first** ("not confident," "unsure," "uncertain") before it looks
for word-rungs like "high." The reason: the substring `"confident"`
appears *inside* `"not confident"`, so a naive "does this text contain the
word high-confidence" check would read "not confident" as a `High` rating —
exactly backwards, and exactly the kind of self-report the gate mechanism
exists to catch. Only after the negation check does it look for word rungs
(checking `"low"` before `"high"`, so a hedge like "low-to-high" resolves
pessimistically), and only after that does it fall back to parsing numbers
— disambiguating a bare `"0.9"` as a probability from a bare `"1"` as a
1-of-10 score.

```ts
// idiomatic TS equivalent — this code does not exist in jcode
type ConfidenceLevel = "low" | "medium" | "high";

function parseConfidence(raw: string): ConfidenceLevel | null {
  const text = raw.trim().toLowerCase();
  if (!text) return null;

  // Negation phrases must be checked BEFORE word-rung substrings —
  // "not confident" contains "confident," which would otherwise
  // false-positive as a High rating.
  const negations = ["not confident", "unsure", "uncertain", "not sure"];
  if (negations.some((phrase) => text.includes(phrase))) return "low";

  if (text.includes("low")) return "low";
  if (text.includes("high")) return "high";
  if (text.includes("medium") || text.includes("moderate")) return "medium";

  // (jcode also falls back to numeric parsing here — 1-10 scores,
  // fractions, percentages — omitted from this sketch for brevity.)
  return null;
}
```

The ordering is the entire lesson: run the negation check first, or a
confident-sounding piece of text about *not* being confident silently
erases the exact signal the rest of the engine depends on.

## The four legal ways to touch the graph

Every mutation function follows the same skeleton: check who's allowed to
do this and whether the graph is in the right state, validate the proposed
change, then commit. Two of the three we'll look at go further and use a
**stage-on-a-clone** pattern: build the new graph on a full clone, run a
cycle check against the clone, and only overwrite the real graph if the
clone is still acyclic.

### `expand_node` — decomposing a node into children

```rust
// jcode: crates/jcode-plan/src/dag/ops.rs:227-367
pub fn expand_node(/* graph, node_id, actor, children */) -> Result<ExpandOutcome, DagError> {
    // 1. ownership + state guard (233-260): reject unless `actor` owns
    //    the node, reject gates decomposing themselves, reject a node
    //    that's already expanded or not currently Running.
    // 2. validate every child id/dependency (264-284): no id collisions,
    //    every depends_on must resolve to a real node.
    let mut staged = graph.clone();               // ops.rs:287
    // 3. insert children, parented to this node, tagged NodeOrigin::Expand
    // 4. deep mode only: auto-insert a gate, kind = parent_kind.gate_kind()
    // 5. flip the parent into a composite join: expanded = true,
    //    status = Queued, append child ids AND the gate id to its own
    //    depends_on
    let cycle = staged.cycle_nodes();
    if !cycle.is_empty() {
        return Err(DagError::WouldCreateCycle(cycle));  // ops.rs:361-364
    }
    *graph = staged;                                // ops.rs:365
    // ...
}
```

Two details worth slowing down for.

First, the **stage-then-commit** shape: nothing mutates the live graph until
a full copy of the proposed result has already passed the cycle check. If
the check fails, the clone is just discarded — the real graph never saw the
bad state. It's the same shape as a database transaction, minus the
database.

Second — and this one is a genuinely non-obvious, verified nuance — in deep
mode the parent node ends up depending on *both* its new children *and* the
auto-inserted gate, even though the gate itself already depends on all the
children. That looks redundant (if the gate is done, the children must
already be done too), but it isn't: the scheduler's dependency check only
cares whether everything is *transitively* done, while the step that
assembles a worker's input reads only a node's *direct* dependencies. If
the child edges were dropped once the gate existed, the eventual synthesis
step would never receive the children's actual output when it wakes back
up. The duplicate edges are load-bearing, not an oversight.

```ts
// idiomatic TS equivalent — this code does not exist in jcode
type Result<T> = { ok: true; value: T } | { ok: false; error: string };

interface Node {
  id: string;
  kind: NodeKind;
  status: NodeStatus;
  owner: string | null;
  parent: string | null;
  dependsOn: string[];
  expanded: boolean;
  isGate: boolean;
}

function expandNode(
  graph: Map<string, Node>,
  nodeId: string,
  actor: string,
  children: { id: string; kind: NodeKind; dependsOn?: string[] }[],
): Result<{ childIds: string[]; gateId: string | null }> {
  const node = graph.get(nodeId);
  if (!node) return { ok: false, error: "unknown node" };
  if (node.owner !== actor) return { ok: false, error: "not owner" };
  if (node.isGate) return { ok: false, error: "gate cannot expand itself" };
  if (node.status !== "running" || node.expanded) {
    return { ok: false, error: "invalid state" };
  }
  if (children.length === 0) return { ok: false, error: "empty children" };

  for (const child of children) {
    if (graph.has(child.id)) return { ok: false, error: `duplicate id ${child.id}` };
    for (const dep of child.dependsOn ?? []) {
      if (!graph.has(dep) && !children.some((c) => c.id === dep)) {
        return { ok: false, error: `unknown dependency ${dep}` };
      }
    }
  }

  // Stage on a copy; never touch the real graph until it passes.
  const staged = new Map(graph);
  const childIds = children.map((c) => c.id);
  for (const child of children) {
    staged.set(child.id, {
      id: child.id,
      kind: child.kind,
      status: "queued",
      owner: null,
      parent: nodeId,
      dependsOn: child.dependsOn ?? [],
      expanded: false,
      isGate: false,
    });
  }

  let gateId: string | null = null;
  // (deep-mode-only gate insertion omitted for brevity — same shape:
  // insert one more node depending on all childIds, tagged isGate: true)

  const parent = { ...node, expanded: true, status: "queued" as const, owner: null };
  parent.dependsOn = [...parent.dependsOn, ...childIds, ...(gateId ? [gateId] : [])];
  staged.set(nodeId, parent);

  if (hasCycle(staged)) {
    return { ok: false, error: "would create a cycle" };
  }
  for (const [id, n] of staged) graph.set(id, n); // commit
  return { ok: true, value: { childIds, gateId } };
}

function hasCycle(graph: Map<string, Node>): boolean {
  const WHITE = 0, GRAY = 1, BLACK = 2;
  const color = new Map<string, number>();
  const visit = (id: string): boolean => {
    color.set(id, GRAY);
    for (const dep of graph.get(id)?.dependsOn ?? []) {
      const c = color.get(dep) ?? WHITE;
      if (c === GRAY) return true; // back edge -> cycle
      if (c === WHITE && visit(dep)) return true;
    }
    color.set(id, BLACK);
    return false;
  };
  for (const id of graph.keys()) {
    if ((color.get(id) ?? WHITE) === WHITE && visit(id)) return true;
  }
  return false;
}
```

### `complete_node` — finishing a node with a typed report

```rust
// jcode: crates/jcode-plan/src/dag/ops.rs:377-411
// 1. ownership + state check (must be Running)
// 2. validate_artifact(mode, node_id, is_gate, &artifact) — ops.rs:400
//    deep mode, non-gate nodes: reject empty `findings`, reject an empty
//    `what_i_did_not_check` list, reject a `confidence` field that doesn't
//    parse to a ConfidenceLevel
// 3. if this node is a gate and mode requires gates:
//    validate_gate_pass(graph, node_id, &artifact) — ops.rs:402 (see below)
// 4. on success: node.status = Done; node.output = Some(artifact);
```

No stage-then-commit dance here — completing a node never changes an edge,
so there's no cycle to guard against. It's a simpler shape: validate
everything about the incoming artifact first, and only after every check
passes does a single field get written. Same idea as a form-submit handler
that refuses to touch application state until the whole payload has passed
validation.

The interesting design decision is in step 2: **light mode and gate
artifacts skip the strict checks entirely**. A gate's artifact is a
pass/fail record about someone else's work, not a scope claim about its
own — so it doesn't need to declare "what I did not check" the way a
regular worker node does.

```ts
// idiomatic TS equivalent — this code does not exist in jcode
interface HandoffArtifact {
  findings: string;
  evidence: string[];
  whatIDidNotCheck: string[];
  confidence?: string;
}

function completeNode(
  graph: Map<string, Node>,
  nodeId: string,
  actor: string,
  mode: Mode,
  artifact: HandoffArtifact,
): Result<void> {
  const node = graph.get(nodeId);
  if (!node) return { ok: false, error: "unknown node" };
  if (node.owner !== actor) return { ok: false, error: "not owner" };
  if (node.status !== "running") return { ok: false, error: "invalid state" };

  if (mode === "deep" && !node.isGate) {
    if (!artifact.findings) return { ok: false, error: "thin artifact: no findings" };
    if (artifact.whatIDidNotCheck.length === 0) {
      return { ok: false, error: "thin artifact: must state what was not checked" };
    }
    if (parseConfidence(artifact.confidence ?? "") === null) {
      return { ok: false, error: "thin artifact: unparseable confidence" };
    }
  }

  // (gate nodes in deep mode additionally run validateGatePass here —
  // see the anti-hallucination section below)

  graph.set(nodeId, { ...node, status: "done" });
  return { ok: true, value: undefined };
}
```

### `inject_from_gate` — a reviewer that doesn't rubber-stamp

```rust
// jcode: crates/jcode-plan/src/dag/ops.rs:444-540
// 1. must be a gate, must be Running, must supply >= 1 new node
// 2. validate new ids/deps, same as expand_node
// 3. stage-then-commit on a clone, same as expand_node
// 4. new nodes are parented to the GATE'S PARENT (ops.rs:505,
//    NodeOrigin::Gap) — they become siblings of whatever the gate
//    was auditing, not children of the gate
// 5. the gate re-queues itself (ops.rs:508-519): status = Queued,
//    owner = None, and the new node ids are appended to the gate's
//    own depends_on — so the gate re-runs once the gap work is done
```

This is `expand_node`'s twin, with one different rule: instead of a review
node finding a problem and walking away, it files new work *and puts itself
back in the queue* to re-check once that work lands. It's the same shape as
a CI pipeline where a failing lint step doesn't just fail the build — it
opens tickets and re-schedules itself to re-run once they close.

```ts
// idiomatic TS equivalent — this code does not exist in jcode
function injectFromGate(
  graph: Map<string, Node>,
  gateId: string,
  actor: string,
  newNodes: { id: string; kind: NodeKind; dependsOn?: string[] }[],
): Result<{ newIds: string[] }> {
  const gate = graph.get(gateId);
  if (!gate) return { ok: false, error: "unknown node" };
  if (!gate.isGate) return { ok: false, error: "not a gate" };
  if (gate.owner !== actor) return { ok: false, error: "not owner" };
  if (gate.status !== "running") return { ok: false, error: "invalid state" };
  if (newNodes.length === 0) return { ok: false, error: "no nodes supplied" };

  const staged = new Map(graph);
  const newIds = newNodes.map((n) => n.id);
  for (const n of newNodes) {
    staged.set(n.id, {
      id: n.id,
      kind: n.kind,
      status: "queued",
      owner: null,
      parent: gate.parent, // sibling of what the gate was auditing
      dependsOn: n.dependsOn ?? [],
      expanded: false,
      isGate: false,
    });
  }
  staged.set(gateId, {
    ...gate,
    status: "queued",
    owner: null,
    dependsOn: [...gate.dependsOn, ...newIds],
  });

  if (hasCycle(staged)) return { ok: false, error: "would create a cycle" };
  for (const [id, n] of staged) graph.set(id, n);
  return { ok: true, value: { newIds } };
}
```

## The worked example: multimonitor support in scrollwm

`docs/SWARM_TASK_GRAPH.md` §9 walks through a scenario end to end — an
agent exploring multimonitor support in a window manager — and it lines up
exactly with the mechanics above. Here's the graph evolving over time:

- **T0 — seed.** The first agent lays down a skeleton, not an answer: one
  `explore` root, a `critique` gate, a `synthesize` node.
- **T1 — expand into facets.** The root decomposes into six sibling
  facets — geometry/layout, hotplug, DPI/scaling, focus/cursor, workspace
  map, existing-code touchpoints — all depending into the same critique
  gate. This is `expand_node`, called once.
- **T2 — fan-out.** The scheduler hands each queued facet with satisfied
  dependencies to a worker.
- **T3 — recursion.** One facet, `hotplug`, turns out to have its own deep
  scope. Its owning worker calls `expand_node` *on the node it owns* — the
  same function, called again, one level down. There is no separate
  "recursion" code path; it's the same validated mutation, applicable at
  any depth by whoever currently owns a node.
- **T4 — atomic facets finish; edges start carrying data.** Four of the six
  facets complete with typed artifacts. The critique gate is still blocked
  — it depends on the still-running hotplug subtree.
- **T5 — reduce.** The hotplug children finish; the same worker that
  expanded hotplug (recorded as its `planner`) wakes back up to synthesize
  one clean hotplug report. The hotplug composite closes.
- **T6 — the gate finds a gap.** The critique gate reads every facet's
  `what_i_did_not_check` list and notices nobody covered "fullscreen on one
  output" or "mixed refresh rate." It calls `inject_from_gate` with two new
  gap nodes and re-queues itself for a second pass. The top-level
  `synthesize` node stays blocked, transitively, through the gate.
- **T7 — gap nodes finish, re-critique passes, synthesis runs.** The final
  synthesis assembles every upstream artifact *by reference* — evidence
  strings pointing at file:line and commit references, not embedded
  content — into the final report.

The design doc's own closing line for this section is worth carrying into
this chapter too: "The graph is never drafted once; it grows wherever depth
or gaps are found and shrinks in attention as subtrees collapse into
synthesized artifacts."

## Gates as jcode's own anti-hallucination mechanism

This is the chapter's central exhibit: `validate_gate_pass`
(`crates/jcode-plan/src/dag/ops.rs:789-878`) is the function that decides
whether a reviewer's "looks good" is actually allowed to close the graph
out. It runs three checks, in this order:

1. **Stale scope.** Every node the gate is auditing must already be `Done`.
   If any are still pending, the pass is rejected outright — this guards
   against the gate racing ahead of work that got added after it started.
2. **Confidence debt.** Any audited node that self-reported
   `ConfidenceLevel::Low` must be explicitly named, by id, in the gate's own
   findings or open questions — or already patched via a prior
   `inject_from_gate` call. Critically, the gate's *own*
   `what_i_did_not_check` field does not count as addressing this. The doc
   comment is blunt about why: "declaring 'I did not check X' is the
   opposite of addressing X."
3. **Coverage debt.** Up to a cap of 20 audited nodes, the passing artifact
   must name *every* done node in scope — not just the shaky ones. "All
   good, no gaps" isn't allowed to pass silently over work it never even
   mentions. Past the cap, full enumeration relaxes, but anything that
   didn't self-report `High` confidence — medium, low, or unparseable —
   still has to be named.

Naming a node "by mentioning its id in text" sounds trivial until you
notice a short id like `"a"` would match almost any English sentence under
a naive substring check. jcode instead uses a hand-rolled word-boundary
scanner (`mentions_node_id`, `ops.rs:593-628`) that treats characters like
`-_.:` as legal id characters but still requires a true boundary on either
side — a small, real correctness bug avoided, not a triviality.

Every failure here comes back as a structured, actionable error — not a
bare "validation failed." The error for unaddressed low confidence, for
instance, names the exact two things the gate could do next: call
`inject_from_gate` with follow-up nodes, or name the id in its own
findings.

```ts
// idiomatic TS equivalent — this code does not exist in jcode
type GateFailure =
  | { ok: false; kind: "stale_scope"; pending: string[] }
  | { ok: false; kind: "unaddressed_low_confidence"; nodes: string[] }
  | { ok: false; kind: "uncovered_siblings"; nodes: string[] };

function validateGatePass(
  graph: Map<string, Node>,
  gateId: string,
  artifact: HandoffArtifact,
): { ok: true } | GateFailure {
  const gate = graph.get(gateId)!;
  const scope = gate.dependsOn.filter((id) => !graph.get(id)?.isGate);

  const pending = scope.filter((id) => graph.get(id)?.status !== "done");
  if (pending.length > 0) return { ok: false, kind: "stale_scope", pending };

  const lowConfidence = scope.filter(
    (id) => graph.get(id)?.output?.confidence === "low",
  );
  const unaddressed = lowConfidence.filter(
    (id) => !mentionsNodeId(artifact.findings, id),
  );
  if (unaddressed.length > 0) {
    return { ok: false, kind: "unaddressed_low_confidence", nodes: unaddressed };
  }

  const uncovered = scope.filter((id) => !mentionsNodeId(artifact.findings, id));
  if (uncovered.length > 0) {
    return { ok: false, kind: "uncovered_siblings", nodes: uncovered };
  }

  return { ok: true };
}

function mentionsNodeId(text: string, id: string): boolean {
  // word-boundary aware, not a bare text.includes(id) —
  // a short id like "a" would otherwise match almost any sentence
  const pattern = new RegExp(`(^|[^\\w.:-])${id.replace(/[.*+?^${}()|[\]\\]/g, "\\$&")}([^\\w.:-]|$)`);
  return pattern.test(text);
}
```

A nice analogy: this is a CI merge gate that programmatically checks a
reviewer's approval comment actually mentions every changed file above some
risk threshold, instead of accepting a bare "LGTM."

## The callback

Here's the honest parallel this chapter has been building toward. jcode
does not let a task node close out on an agent's self-report that it's
"done" and "fine." A second, adversarial node — the gate — is structurally
required to re-examine the completed work, specifically hunting for what
the first agent admitted it didn't check. And the *engine itself*, not just
a prompt instructing it to be careful, refuses to let that gate rubber-stamp
past an unaddressed low-confidence node or a sibling it never mentioned.

That's the same shape as the process that produced this handbook. A
cite-and-verify pass re-opens every source file a draft claims something
about, and a claim doesn't get accepted just because it's stated more
confidently the second time — it gets accepted because someone (or
something) went and checked. The correction at the top of this chapter
wasn't a special case. It was the same mechanism, one level up, catching
the same kind of mistake jcode's own gates exist to catch.
</content>
