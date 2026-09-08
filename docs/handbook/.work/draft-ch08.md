# Chapter 8 — Design-Pattern Glossary & Further Reading

You've now seen jcode's swarm machinery from several angles: spawning,
fan-out/fan-in, cancellation, shared state, the task DAG, communication.
This chapter is the cheat sheet — pattern name, where it lives, the Rust
primitive, and the TS primitive you'd reach for instead. Use it as an index
back into the codebase, not a substitute for the chapters.

Two notes before you scan it. Not every pattern has a clean TS
equivalent — a few exist specifically because Rust gives you real
OS-thread parallelism and single-threaded TS doesn't, and the table says so
rather than forcing a false match. And `dag::Mode` (deep/light task-graph
rigor, Chapter 6) and `SwarmSpawnMode` (visible/headless/inline/auto window
rendering, row 15) are two unrelated concepts that happen to share a
name — don't conflate them.

## Fan-out / fan-in combinators

| Pattern | jcode file:line | Rust primitive | TS equivalent |
|---|---|---|---|
| Planner → fan-out → fan-in → integrate¹ | `swarm.rs:1615-1701`, fan-in at `1671` | `try_join_all` — all-or-nothing, ordered `Vec<T>` | `Promise.all(tasks)` — near-exact match |
| Concurrent tool calls, incremental progress | `tool/batch.rs:282-295`, drain `300-317` | `FuturesUnordered` — yields completion order, not submission order; re-sorted after | No exact match; closest is an async generator yielding as tasks settle. Contrast `Promise.all`, which yields once, at the end |
| Event-driven fan-in via broadcast + select | `comm_await.rs:209-303`, `select!` `271-300` | `broadcast::Receiver::recv()` — can return `Lagged(n)` if a slow receiver falls behind the ring buffer | No built-in match for "lossy pub/sub with an explicit lag signal"; closest is an `EventEmitter` whose slow consumer silently drops events — Rust tells you |
| any/all wait mode, live-updating not one-shot | `comm_await.rs:95-126` | `Iterator::any`/`all` re-evaluated inside a polling loop | Targets `Promise.race`/`all`, but as a repeatedly-checked predicate over shared state, not a one-shot combinator |

¹ Debug-tooling path only (`debug_command_exec.rs:132-139`, `debug_jobs.rs:77-95`) — not reachable via the live `swarm` tool; see Chapter 2/3.

## Cancellation & lifecycle

| Pattern | jcode file:line | Rust primitive | TS equivalent |
|---|---|---|---|
| Fire-and-forget background task | `comm_session.rs:772-822` | `tokio::spawn`, `JoinHandle` dropped, never awaited | An `async function` called without `await` — a "floating promise"; errors only surface via a side effect |
| Structured concurrency scope, deadlock-safe shutdown | `RuntimeTaskScope`, `runtime.rs:33-79` | `JoinSet<()>` + `CancellationToken` (hierarchical via `child_token()`); shutdown drains the `JoinSet` from under its `Mutex` before awaiting, avoiding a lock-across-await deadlock | `AbortController` threaded into every spawned op, plus a `Set<Promise>` awaited via `Promise.allSettled` after `abort()` |
| Cooperative, not preemptive, cancellation | `runtime.rs:164-167`/`376-379`/`432-435` | `select!` racing work against `cancellation.cancelled()`; nothing preempts a running task | Exactly `AbortSignal` semantics: code must check `signal.aborted` or use an abortable API |
| Async-aware interrupt flag, lost-wakeup-safe | `InterruptSignal`, `jcode-agent-runtime/src/lib.rs:32-117` | `AtomicBool` + `Notify` (async condvar) + an `AtomicU64` epoch so a racing `fire()` is never lost | Mostly moot — single-threaded JS has no gap between "check aborted" and "register a listener." A "Rust is genuinely harder here" callout, not a port target |

## Shared state & singletons

| Pattern | jcode file:line | Rust primitive | TS equivalent |
|---|---|---|---|
| `Arc<RwLock<HashMap<...>>>` registries | `state.rs:108-113`, `runtime.rs:90-120` (double-locking: registry under `RwLock`, each `Agent` under its own `Mutex`) | `Arc` (refcount) + `RwLock` (many readers OR one writer) | No real equivalent in single-threaded JS. Either careful `await` placement replaces locking, or genuine `worker_threads` with message-passing to one owning thread (more idiomatic than `SharedArrayBuffer`) |
| Process-wide lazy singleton behind a lock | `Bus::global()`, `bus.rs:499-502`; also `swarm.rs:108-113`, `state.rs:15-38` | `OnceLock`/`LazyLock` + `get_or_init` | `let instance; export function getBus() { if (!instance) instance = new Bus(); return instance; }` — Node's module cache gives "once" almost free |
| Debounce instead of locking harder | `swarm.rs:742-812`, `bus.rs:542-606` (750ms window) | State (`last_published_at`) checked before every publish | A debounce/coalesce wrapper (`lodash.debounce` or hand-rolled `setTimeout`) around `.emit()` |

## Pub/sub & event delivery

| Pattern | jcode file:line | Rust primitive | TS equivalent |
|---|---|---|---|
| Global broadcast bus, closed tagged event vocabulary | `BusEvent` (~30 variants), `bus.rs:394-466`; `channel(256)` at `512` | `tokio::sync::broadcast` — each subscriber gets its own cursor into one bounded ring buffer; slow subscribers lag independently | A process-wide `EventEmitter` with a closed, documented event-name set (ideally a discriminated union). No built-in bounded-buffer/lag signal — that part is genuinely tokio-specific |
| Targeted unicast-per-member fan-out (vs. the bus) | `swarm.rs:836-920`, delivery loop `899-908` | Per-connection `mpsc::UnboundedSender`, fanned out manually | Iterating a `Map<sessionId, WebSocket>` and `.send()`-ing to each match — targeted, not a shared topic |
| Replay buffer beside a broadcast channel that has none | `event_history`, `runtime.rs:107`, populated by `swarm.rs:1233-1257` | A bounded `VecDeque` behind a lock, appended alongside every broadcast send | A ring-buffer array (`push`/`shift` at length N) kept beside an `EventEmitter` |
| Bidirectional two-map-mirror index | `ChannelIndex`, `jcode-swarm-core/src/lib.rs:235-340` | Two nested `HashMap`s kept in lockstep for near-O(1) lookups both directions | `Map<string, Map<string, Set<string>>>` pair, same dual-index, same manual-sync discipline |

## Indexing & authorization

| Pattern | jcode file:line | Rust primitive | TS equivalent |
|---|---|---|---|
| Spawn ancestry via parent-pointer walk (no stored tree) | `swarm.rs:41-60` | Iterative lookup over `HashMap<String, SwarmMember>`, cycle-guarded with a visited set, following `report_back_to_session_id` | A `Map<string, Member>` plus a `while` loop up a `reportsTo` field, with a `Set<string>` visited-guard |
| Authorization via ancestry (subtree ownership) | `swarm.rs:76-85` | Boolean check reusing the ancestry walk above | Same `Map` walk-up used as an auth predicate before a stop/control action |
| Mode-gated spawn rendering — **not `dag::Mode`** | `SwarmSpawnMode`, `jcode-config-types/src/lib.rs:646-657` | `#[derive(Default)]` enum: `Visible`/`Headless`/`Inline` (default)/`Auto` | A string-literal union with a default parameter value |

## API / argument shape

| Pattern | jcode file:line | Rust primitive | TS equivalent |
|---|---|---|---|
| Layered optional-parameter delegation chain | `swarm.rs:1291-1527`ish, each `#[expect(clippy::too_many_arguments)]` documenting the tradeoff | Layered free functions, each widening the parameter list by one field | An options-object pattern (`opts?: { report?; tldr? }`) — TS sidesteps this for language reasons, not because the Rust is wrong |
| Structured, actionable validation errors | `DagError` + `Display`, `dag/mod.rs:442-531` (11 variants) | An error whose message tells the caller the exact next step | A discriminated-union `Result` error whose formatter gives the same actionable text, not a bare "validation failed" |

## The task DAG's own idioms (Chapter 6)

| Pattern | jcode file:line | Rust primitive | TS equivalent |
|---|---|---|---|
| Validated core wrapped by a stateful adapter ("lift, apply, lower") | `jcode-plan/src/bridge.rs:1-9`, `to_task_graph`/`apply_task_graph` at `94`/`124` | Free functions lifting a live `VersionedPlan` into a pure `TaskGraph`, applying an op, lowering back | `liftToGraph`/`applyOp`/`lowerToPlan` — arguably the most portable lesson here |
| Stage-on-a-clone, validate, commit-or-reject | `dag/ops.rs`: `expand_node` (`227-367`, clone `287`, commit `365`), `inject_from_gate` (`444-540`) | Clone the graph, mutate the clone, run `cycle_nodes()`, only reassign if valid | Build a new object (spread/immer), validate it, only then reassign the "real" state reference |
| Encapsulation via a private field + validated methods | `TaskGraph.nodes: Vec<TaskNode>` private, `dag/mod.rs:539-542` | No `pub` on the field — mutation forced through `ops::` | A class with a private field and only getter/mutator methods exposed |
| Feature flag reduced to one predicate function | `dag::Mode`/`requires_gates()`, `dag/mod.rs:37-49` | Consulted inside `expand_node`/`complete_node`, never scattered as ad hoc checks | A single `requiresGates(mode): boolean` used at both call sites |
| Lenient free-text-to-enum parsing, substring-order traps | `ConfidenceLevel::parse`, `dag/mod.rs:141-204` | Negation phrases checked before word-rung substrings, so "not confident" can't match "confident" → High | `parseConfidence(raw)` with the same negation-first ordering |
| Word-boundary-safe substring matching | `mentions_node_id`, `ops.rs:593-628` | A hand-rolled boundary scanner treating `-_.:` as legal id characters | A `RegExp` with `\b`, or a hand-rolled scanner where punctuation breaks regex `\b` |
| Side-map extension instead of widening the base record | `NodeMeta`, `jcode-plan/src/lib.rs:117-146`, keyed in `VersionedPlan.node_meta` | A companion map keyed by id, instead of optional fields only one subsystem needs | `Map<TaskId, NodeMeta>` kept beside `Task[]` |

## The agent-facing protocol surface

| Pattern | jcode file:line | Rust primitive | TS equivalent |
|---|---|---|---|
| Prompt text as the LLM-facing protocol, separate from the Rust API | `jcode-swarm-core/src/lib.rs:390-433`, `449-499` | Functions generating `<system-reminder>` text naming the only legal turn-ending actions | No real TS "pattern" — the tool schema is the enforced contract, but the natural-language directive is what gets the model to use it correctly; the two must be kept in sync by hand |
| The verified tool-call chain: prompt → schema → match arm → wire request → handler → engine | schema at `tool/communicate.rs:1959` → match arm `2629-2656` → `server/comm_graph.rs:339-402` → `dag::expand_node` at line `368` | A typed request enum dispatched through a server handler into a pure engine function | A typed RPC dispatcher (`Record<ActionName, (params) => Promise<Result>>`) whose handlers call pure domain functions |

One caveat: the exact wire-JSON shape of `Request::CommExpandNode` and
friends, as serialized in `jcode-protocol`, wasn't independently opened
during this handbook's research. The Rust-side chain above — prompt text
through the `dag::` function call — is fully verified; the literal wire
format is an open question if you go looking.

## Further reading

- **`docs/SWARM_ARCHITECTURE.md`** (318 lines) — the older, agent-first
  design doc, explicitly "largely implemented" but superseded in framing by
  the task-graph doc below. Best source for vocabulary and lifecycle: roles
  and mode-gated spawning (22-58), lifecycle states and notifications
  (89-106), completion report policy (108-124), communication (194-254),
  and the "no locks" language (307-311) that Chapter 5 adds nuance to. One
  flag: its 8-state lifecycle list doesn't map 1:1 to the code's actual
  `SwarmLifecycleStatus` enum (13 variants plus `Other`,
  `jcode-swarm-core/src/lib.rs:136-151`) — read the doc as intent, the enum
  as current ground truth.
- **`docs/SWARM_TASK_GRAPH.md`** (605 lines) — the newer, DAG-first doc
  Chapter 6 draws from most heavily. Worth reading directly: §1a's Deep vs.
  Light table (54-98), §5's dataflow description (195-216), §6.4's
  "implemented enforcement" section (275-304, the richest passage tying
  prose to real `DagError` variants), §8a's communication-rework notes
  (407-475), and §9, the worked example Chapter 6 walks through in full
  (478-549).
- **`crates/jcode-plan/src/dag/`** — the DAG engine: `mod.rs` (types),
  `ops.rs` (validated mutations), `schedule.rs` (ready-set, dispatch,
  dataflow hydration), `tests.rs` (1392 lines — the best source for every
  edge case the `DagError` variants exist to catch). `sim.rs` (155 lines, a
  deterministic simulator) wasn't opened during this handbook's research —
  a genuine go-read-it-yourself pointer.
- **`crates/jcode-app-core/src/server/`** — `swarm.rs` (member registry,
  plan-driven fan-out), `comm_session.rs` (spawn path), `comm_await.rs`
  (fan-in wait), `comm_graph.rs` (DAG tool-call handlers), `state.rs`
  (`SwarmState`/`SwarmMember`).
- **`crates/jcode-app-core/src/tool/batch.rs`** and **`communicate.rs`** —
  the intra-agent batch tool and the `swarm` tool's full action surface
  (roughly 30 actions; this handbook covers a handful).
- **`crates/jcode-base/src/bus.rs`** — the global event bus and its full
  `BusEvent` vocabulary (~30 variants, 394-466), most of which this
  handbook doesn't individually cover.
- **`crates/jcode-agent-runtime/src/lib.rs`** — `InterruptSignal` and its
  five regression tests (142-283), pinning down the lost-wakeup race from
  Chapter 4.

Everything in this handbook cites back to one of these files or docs — if a
claim here doesn't line up with what you find when you open them, the code
is the ground truth, not the prose.
</content>
