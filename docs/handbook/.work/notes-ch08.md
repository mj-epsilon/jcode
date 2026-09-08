# Phase 2 Analysis Notes — Chapter 8: Design-Pattern Glossary & Further Reading

Purpose per the plan: "a one-page cheat sheet mapping pattern name -> jcode
file -> Rust primitive -> TS primitive equivalent," listing patterns that
Chapters 1-7 already established (not re-deriving new ones), plus pointers
into the two design docs and the crates for readers who want to go deeper.

All file:line citations below are drawn from the Phase 1 map's already-
verified entries. Where I additionally opened the source myself this pass
(to backstop a citation I intended to feature prominently in the table), I
mark it "(re-verified)". Nothing here is invented; every row traces to a
map citation or my own read.

---

## 1. The cheat-sheet table (raw rows for Phase 3 to format)

Columns: **Pattern** | **jcode file:line** | **Rust primitive** | **TS/Node
idiomatic equivalent** | **Source chapter**

1. **Spawn ancestry via parent-pointer walk (no stored tree)**
   `crates/jcode-app-core/src/server/swarm.rs:41-60` (`swarm_ancestors`) —
   walks `report_back_to_session_id` upward, cycle-guarded with a visited
   set. Rust primitive: plain recursive/iterative lookup over a
   `HashMap<String, SwarmMember>` behind `Arc<RwLock<_>>`. TS equivalent: a
   `Map<string, Member>` plus a `while` loop following a `reportsTo` field,
   with a `Set<string>` visited-guard. Chapter 2.

2. **Mode-gated spawn rendering** — `SwarmSpawnMode` enum (`Visible`,
   `Headless`, `Inline` default, `Auto`),
   `crates/jcode-config-types/src/lib.rs:646-657`. Rust primitive: a
   `#[derive(Default)]` enum with doc-commented variants. TS equivalent: a
   string-literal union type (`"visible"|"headless"|"inline"|"auto"`) with
   a default parameter value. Chapter 2. **Do not confuse with `dag::Mode`
   (item 15) — two unrelated "mode" axes in the same codebase.**

3. **Fire-and-forget background task with a caveat** —
   `crates/jcode-app-core/src/server/comm_session.rs:772-822`
   (`spawn_swarm_agent`'s conditional `tokio::spawn(async move { ... })`).
   Rust primitive: `tokio::spawn` whose `JoinHandle` is dropped, not
   awaited or stored. TS equivalent: calling an `async function` without
   `await` (a "floating promise" — normally an ESLint
   `no-floating-promises` violation); worth the explicit callout that
   errors from this task are only observable via a side effect
   (`update_member_status_with_report`), never via a caller-visible
   `Result`. Chapter 2, 4.

4. **Authorization via ancestry (subtree ownership)** —
   `swarm.rs:76-85` (`swarm_is_self_or_ancestor`). Rust primitive: boolean
   check reusing the ancestry-walk primitive (item 1). TS equivalent: same
   `Map` + walk-up-the-chain check used as an authorization predicate before
   allowing a stop/control action. Chapter 2.

5. **Planner -> fan-out -> fan-in -> integrate (three sequential agent
   calls around one concurrent middle step)** — `swarm.rs:1615-1701`
   (`run_swarm_message`), fan-out at 1657-1670, fan-in
   `try_join_all(task_futures).await?` at **line 1671**. Rust primitive:
   `futures::future::try_join_all` — run N futures concurrently, resolve
   with `Vec<T>` if all succeed, reject on first error. TS equivalent:
   `Promise.all(tasks)` (near-exact semantic match: both are
   all-or-nothing, both resolve to an ordered array). Chapter 3.

6. **Intra-agent concurrent tool execution with incremental progress** —
   `crates/jcode-app-core/src/tool/batch.rs:282-295`
   (`FuturesUnordered` construction) and the drain loop
   `while let Some(...) = stream.next().await` at lines 300-317. Rust
   primitive: `futures::stream::FuturesUnordered` — poll a group of
   in-flight futures, yield whichever finishes next, in completion order
   (not submission order); results re-sorted back to input order afterward
   (`results.sort_by_key`, line 319). TS equivalent: no single built-in
   matches this shape exactly — closest is manually racing an array of
   promises (repeatedly `Promise.race` over the remaining set) or an async
   generator that yields as each task settles; contrast explicitly against
   `Promise.all`, which only yields once, at the very end. Chapter 3.

7. **Event-driven fan-in via broadcast + select, degrading gracefully on
   backpressure** — `crates/jcode-app-core/src/server/comm_await.rs:
   209-303`, `tokio::select!` at 271-300 racing `sleep_until(deadline)`
   (272) against `event_rx.recv()` (277). Rust primitive:
   `tokio::sync::broadcast::Receiver::recv()`, which can return
   `Err(Lagged(n))` if the receiver fell behind a bounded ring buffer (the
   sender never blocks for slow receivers — it overwrites). The code
   treats a lag as non-fatal because shared state, not the event stream, is
   the source of truth (comment at comm_await.rs:284-287). TS equivalent:
   nothing built-in matches "lossy bounded pub/sub with an explicit lag
   signal" exactly; closest teaching analogy is an `EventEmitter` where a
   slow consumer's queue could silently drop old events, except Rust's
   broadcast channel tells the receiver explicitly via `Lagged(n)` instead
   of silently dropping. Chapter 3, 7.

8. **any/all wait semantics as an explicit mode, not a combinator call** —
   `comm_await.rs:95-126` (`completion_mode`/`mode_satisfied`/
   `mode_summary`) — `"any"` uses `.iter().any(...)`, `"all"` (default)
   uses `.iter().all(...)`. Rust primitive: `Iterator::any`/`Iterator::all`
   inside a polling loop, not a single blocking combinator. TS equivalent:
   the semantic targets are `Promise.race` (any) and `Promise.all` (all),
   but implemented here as a repeatedly-re-evaluated predicate over shared
   state rather than a single combinator over a fixed future set — worth
   drawing the contrast explicitly (this is a *live-updating* any/all, not
   a one-shot one). Chapter 3.

9. **Structured concurrency scope: one cancellation token owns every
   spawned task, and shutdown drains the registry first to avoid a
   lock-held-across-await deadlock** — `RuntimeTaskScope`,
   `crates/jcode-app-core/src/server/runtime.rs:33-79` (re-verified
   directly this pass; struct at 33-37, `spawn` at 39-59, `shutdown` at
   61-74 exactly as cited). Rust primitives: `tokio::task::JoinSet<()>`
   (spawn many, join/collect as a group) + `tokio_util::sync::
   CancellationToken` (a shareable "please stop" flag; `child_token()`
   gives a linked token that also fires when the parent fires — like
   `AbortSignal` but explicitly hierarchical). `shutdown`'s
   `std::mem::take(&mut *owned_tasks)` (runtime.rs:69) drains the `JoinSet`
   out from under its `Mutex` *before* awaiting each child, specifically so
   a task that's mid-registration under the same mutex can't deadlock
   against the shutdown routine — a subtle, real concurrency-bug class
   worth featuring, not just the happy path. TS equivalent: an
   `AbortController` whose `signal` is threaded into every spawned async
   operation, plus a `Set<Promise>` (or an array) tracking in-flight work
   that `shutdown()` awaits via `Promise.allSettled` after first calling
   `abort()`. Chapter 4.

10. **Cooperative, not preemptive, cancellation** —
    `runtime.rs:164-167`/`376-379`/`432-435`, `tokio::select!` racing real
    work against `cancellation.cancelled()`. Rust primitive:
    `tokio::select!` — proceed with whichever branch finishes first,
    drop/cancel the rest. Cancellation only takes effect at an `.await`
    point that's actually racing the token; nothing preempts a running
    task. TS equivalent: exactly `AbortSignal` semantics — code must check
    `signal.aborted` or pass the signal into an abortable API (`fetch`,
    etc.); it is not like killing a thread, same model as Rust here.
    Chapter 4.

11. **Hand-rolled async-aware interrupt flag solving a lost-wakeup race**
    — `InterruptSignal`, `crates/jcode-agent-runtime/src/lib.rs:32-117`.
    Rust primitives: `AtomicBool` (lock-free sync read/write) +
    `tokio::sync::Notify` (async wake-one-or-all-waiters, i.e. an async
    condition variable) + an `AtomicU64` epoch counter so a racing `fire()`
    between a caller's epoch-check and its `reset` is never silently lost
    (`reset_if_epoch`, lines 78-90; regression-tested as "issue #428" by 5
    tests, lines 142-283). TS equivalent: mostly moot — single-threaded
    JS/TS sidesteps the specific race this file exists to solve, because
    there's no gap between "check if aborted" and "register a listener"
    when only one thread can ever be running your code at a time. Good
    explicit "Rust is meaningfully more complex here, and here's exactly
    why (real OS-thread parallelism)" chapter callout, not a like-for-like
    port target. Chapter 4.

12. **Arc<RwLock<HashMap<...>>> — shared, many-readers-or-one-writer
    registries** — `crates/jcode-app-core/src/server/state.rs:108-113`
    (`SwarmState`: `members`, `swarms_by_id`, `plans`, `coordinators`, each
    independently `Arc<RwLock<_>>`), and `runtime.rs:90-120`
    (`ServerRuntime`'s `sessions: Arc<RwLock<HashMap<String,
    Arc<Mutex<Agent>>>>>` — note the double-locking: registry behind
    `RwLock`, each `Agent` behind its own `Mutex`). Rust primitives: `Arc`
    (atomic refcount, many owners) + `RwLock` (many concurrent readers OR
    one exclusive writer, vs. `Mutex`'s always-exclusive). TS equivalent:
    none of this exists in single-threaded JS — a `Map`/object is
    inherently touched by only one turn of the event loop at a time. The
    honest translation is either (a) cooperative concurrency: one process,
    async functions that never truly run simultaneously, so careful
    `await` placement replaces locking, or (b) genuine `worker_threads`
    parallelism, which needs `SharedArrayBuffer`+`Atomics` for true shared
    memory (rare) or message-passing to a single owning thread (far more
    common, and the more idiomatic TS teaching answer). Chapter 5.

13. **Process-wide lazy singleton behind a lock** — three independent
    instances of the same idiom: `Bus::global()`,
    `crates/jcode-base/src/bus.rs:499-502` (re-verified directly this
    pass):
    ```rust
    pub fn global() -> &'static Bus {
        static INSTANCE: OnceLock<Bus> = OnceLock::new();
        INSTANCE.get_or_init(Bus::new)
    }
    ```
    plus `swarm.rs:108-113` (`pending_swarm_status_broadcasts()`, an
    `OnceLock<StdMutex<HashMap<...>>>`) and `state.rs:15-38`
    (`BACKGROUND_TOOL_SIGNALS`, a `LazyLock<StdMutex<HashMap<String,
    InterruptSignal>>>`). Rust primitive: `std::sync::OnceLock`/`LazyLock`
    + `get_or_init`. TS equivalent: a module-level `let instance;
    export function getBus() { if (!instance) instance = new Bus(); return
    instance; }` — process-wide singleton, not tied to any one
    request/session; Node module caching gives you the "only ever
    initialized once" property almost for free, without needing an
    explicit lock. Chapter 5, 7.

14. **Debounce instead of locking harder** — two independent
    implementations of the same idea: `swarm.rs:742-812`
    (`broadcast_swarm_status`, debounced once swarm size passes a
    threshold) and `crates/jcode-base/src/bus.rs:542-606`
    (`Bus::publish_models_updated`, fixed 750ms window,
    `MODELS_UPDATED_DEBOUNCE` — re-verified directly this pass at
    `bus.rs:476`). Rust primitive: a small piece of state (`last_published_
    at`/`publish_pending`) tracked alongside the broadcast sender, checked
    before every publish. TS equivalent: a classic debounce/coalesce
    wrapper (`lodash.debounce` or a hand-rolled `setTimeout`-based
    coalescer) around an event-emitter's `.emit()` call — same idea,
    different runtime. Chapter 5.

15. **Bidirectional two-map-mirror index** — `ChannelIndex`,
    `crates/jcode-swarm-core/src/lib.rs:235-340` (re-verified directly this
    pass: `by_swarm_channel: HashMap<String, HashMap<String,
    HashSet<String>>>` and `by_session: HashMap<String, HashMap<String,
    HashSet<String>>>`, kept in sync by `subscribe`/`unsubscribe`/
    `remove_session`). Rust primitive: two nested `HashMap`s maintained in
    lockstep so both "who's in this channel" and "what channels is this
    session in" are near-O(1) lookups instead of a full scan. TS
    equivalent: a `Map<string, Map<string, Set<string>>>` pair, same
    dual-index shape, same manual-sync discipline. Chapter 7.

16. **Single global broadcast bus with a closed, tagged event vocabulary**
    — `BusEvent` enum (~30 variants), `crates/jcode-base/src/bus.rs:
    394-466`; `Bus` struct is just `sender: broadcast::Sender<BusEvent>`
    plus small debounce state (re-verified: `bus.rs:468-474`); fixed
    256-slot ring buffer (`broadcast::channel(256)`, `bus.rs:512`,
    re-verified). Rust primitive: `tokio::sync::broadcast` — every
    subscriber gets its own read cursor into the same bounded ring buffer;
    slow subscribers lag independently rather than blocking fast ones or
    the publisher. TS equivalent: a single process-wide `EventEmitter`
    (Node) with a documented, closed set of event names/payload shapes
    (ideally a discriminated union of payloads, mirroring the Rust enum) —
    the "everything crosscutting flows through one pub/sub" idea, though
    Node's `EventEmitter` has no built-in bounded-buffer/lag-signal
    behavior (that part is genuinely Rust/tokio-specific, see item 7).
    Chapter 7.

17. **Targeted unicast-per-member fan-out, contrasted with the global
    broadcast bus** — `swarm.rs:836-920`
    (`broadcast_swarm_plan_with_previous`), delivery loop at lines 899-908
    sends `ServerEvent::SwarmPlan` to each participant's own
    `member.event_tx` (a per-member channel), not a shared broadcast
    channel. Rust primitive: per-connection `mpsc::UnboundedSender`,
    fanned out to manually in a loop. TS equivalent: iterating a
    `Map<sessionId, WebSocket>` and calling `.send()` on each matching
    connection individually — plain targeted delivery, not a shared
    topic/broadcast. Chapter 7. Good "not every fan-out pattern in this
    codebase is the same primitive" contrast point for the chapter.

18. **Replay/history buffer alongside a broadcast channel that has none** —
    `event_history: Arc<RwLock<VecDeque<SwarmEvent>>>` (referenced in
    `runtime.rs:107` per Phase 1; populated by `swarm.rs`'s
    `record_swarm_event`, lines 1233-1259). Rust primitive: a bounded
    `VecDeque` behind a lock, manually appended to alongside every
    broadcast send, so late subscribers can catch up on recent history —
    something `tokio::sync::broadcast` does not provide on its own. TS
    equivalent: a ring-buffer array (`events.push(e); if (events.length >
    N) events.shift();`) maintained alongside an `EventEmitter`, exactly
    the same "the pub/sub primitive has no memory, so bolt on a small
    buffer next to it" idea. Chapter 7.

19. **Three-layer optional-parameter delegation chain (each layer adds one
    more piece of context)** — `update_member_status` (swarm.rs:1291-1313)
    -> `update_member_status_with_report` (1319-1343) ->
    `update_member_status_with_report_tldr` (1349-1527ish), each wrapper
    carrying an `#[expect(clippy::too_many_arguments, reason = "...")]`
    attribute documenting the tradeoff as deliberate, not an oversight.
    Rust primitive: layered free functions, widening the parameter list one
    field at a time. TS equivalent: an options-object pattern
    (`updateMemberStatus(id, status, opts?: { report?: string; tldr?:
    string })`) — TS/JS sidesteps the "too many positional arguments"
    problem entirely via named/optional object parameters, worth noting as
    a case where the idiomatic TS answer is simpler than the Rust original
    for structural language reasons, not because the Rust code is wrong.
    Chapter 3, 7.

20. **Dead-worker salvage: eager reclaim of a dead member's assignments** —
    `DeadMemberSalvage` struct (swarm.rs:294-300),
    `salvage_plan_assignments_of` (344-390),
    `salvage_assignments_of_dead_member` (396-449). Rust primitive: a sync
    function mutating a `VersionedPlan` in place, wrapped by an async
    persist+broadcast+notify caller — the "pure mutation, impure shell"
    split appears again here at a smaller scale than the DAG engine (item
    21). TS equivalent: a pure `salvagePlanAssignments(plan, deadMemberId):
    Plan` function, called from an async orchestrator that persists and
    broadcasts the result — same split. Chapter 3, 7.

21. **Validated mutation core wrapped by a stateful adapter shell ("lift,
    apply, lower")** — `crates/jcode-plan/src/bridge.rs:1-9` doc comment
    (quoted in Chapter 6 notes): the `dag` engine has no side effects and
    no knowledge of the server; `bridge.rs` lifts the server's live
    `VersionedPlan` into the engine's `TaskGraph`, applies a pure engine
    function, lowers the result back. Rust primitive: free functions
    `to_task_graph`/`apply_task_graph` (`bridge.rs:94`, `124`) around a
    pure, cloneable `TaskGraph` value type. TS equivalent: exactly "keep
    your core logic as pure functions over plain data, isolate all I/O and
    mutation in a thin adapter layer around it" — a `liftToGraph(plan):
    Graph` / `applyOp(graph, op): Graph` / `lowerToPlan(plan, graph): Plan`
    triple, the single most portable architectural lesson in the whole
    handbook. Chapter 6.

22. **Stage-on-a-clone, validate, commit-or-reject (optimistic mutation
    with a rollback-by-discarding-the-clone)** —
    `crates/jcode-plan/src/dag/ops.rs`: `expand_node` (227-367,
    `staged = graph.clone()` at 287, commit at 365),
    `inject_from_gate` (444-540, same shape). Rust primitive: `TaskGraph:
    Clone`, mutate the clone, run `cycle_nodes()`, only assign `*graph =
    staged` if valid. TS equivalent: build a new object (structural-sharing
    copy, e.g. via spread/immer) representing the proposed state, validate
    it, and only then reassign the "real" state reference — the same
    "propose, validate, commit" shape as a database transaction, without
    needing an actual database. Chapter 6.

23. **Encapsulation via a private field + a validated method surface** —
    `TaskGraph.nodes: Vec<TaskNode>` is private (`dag/mod.rs:539-542` —
    accessed only through `get`/`get_mut`/`push`/etc.), enforcing that all
    mutation flows through `ops::` functions and never splices the node
    list directly. Rust primitive: no `pub` on the field. TS equivalent: a
    class with a private (`#`-prefixed or TS `private`) field and only
    getter/mutator methods exposed — same encapsulation idea, native to
    both languages. Chapter 6.

24. **A boolean feature-flag reduced to a single predicate function,
    consulted at exactly the decision points that matter** — `dag::Mode`
    (`Deep`/`Light`) and `Mode::requires_gates()`
    (`dag/mod.rs:37-49`), consulted inside `expand_node` (whether to
    auto-insert a gate) and `complete_node` (whether artifact validation is
    strict) — never scattered as ad hoc `if mode == Deep` checks elsewhere.
    TS equivalent: a single `requiresGates(mode): boolean` function used at
    both call sites, instead of comparing a mode string/enum inline at N
    places — the general "centralize a flag's meaning in one function"
    lesson. Chapter 6.

25. **Lenient free-text-to-enum parsing, careful about substring/ordering
    traps** — `ConfidenceLevel::parse`, `dag/mod.rs:141-204` (negation
    phrases checked before word-rung substrings, specifically to avoid
    "not confident" matching "confident" -> High). TS equivalent: a
    `parseConfidence(raw: string): "low"|"medium"|"high"|null` with the
    same negation-first ordering — a compact, reproducible bug-class
    demo for "substring parsing order matters." Chapter 6.

26. **Word-boundary-safe substring matching (why naive `.includes()` is a
    real correctness bug here)** — `mentions_node_id`, `ops.rs:593-628`.
    TS equivalent: a `RegExp` with `\b` word boundaries, or a hand-rolled
    boundary scanner if the id alphabet includes characters (like `.`/`:`)
    that don't line up with regex `\b`'s definition of a word character.
    Chapter 6.

27. **Structured, actionable validation errors (rejection messages that
    tell the caller exactly what to do next)** — `DagError` enum + its
    `Display` impl, `dag/mod.rs:442-531` (11 variants, e.g.
    `UnaddressedLowConfidence` at 460, whose error string names the exact
    two remedies: `inject_gap` with follow-ups, or name the id in
    findings). TS equivalent: a discriminated-union `Result` error type
    (`{ kind: "unaddressed_low_confidence", gate, nodes }`) whose consumer-
    facing message (or a formatter over it) gives the same actionable
    next-step text, rather than a bare "validation failed." Chapter 6 —
    this is also the chapter's explicit callback to the handbook
    pipeline's own MISCITED/UNVERIFIABLE review-gate findings format.

28. **Side-map extension instead of widening the base record type** —
    `NodeMeta`, `crates/jcode-plan/src/lib.rs:117-146`, stored as
    `VersionedPlan.node_meta: HashMap<String, NodeMeta>` keyed by plan-item
    id, alongside `SwarmTaskProgress`'s `task_progress` side-map
    (lib.rs:36-75) using the identical shape. TS equivalent: `Map<TaskId,
    NodeMeta>` kept beside `Task[]`, instead of adding a growing pile of
    optional fields to `Task` itself for a feature only one subsystem
    needs. Chapter 6.

29. **Prompt-text as the LLM-facing protocol surface, separate from the
    Rust API surface** — `append_deep_node_instructions`/
    `append_deep_gate_instructions`,
    `crates/jcode-swarm-core/src/lib.rs:390-433`, `449-499` (re-verified
    directly this pass, full bodies read) — these functions literally
    generate the `<system-reminder>` text naming `expand_node`/
    `complete_node`/`inject_gap` as the only two legal ways for a worker
    or gate to end its turn; idempotent via `SWARM_DEEP_NODE_MARKER`. No
    real TS equivalent as a "pattern" — this is closer to a general lesson
    for anyone building tool-using agents: the tool schema (item 30) is
    the enforced contract, but the natural-language directive is what
    actually gets the model to use it correctly, and both need to stay in
    sync by hand. Chapter 6, 7.

30. **The verified tool-call chain: prompt text -> JSON Schema enum ->
    typed match arm -> typed wire request -> server handler -> engine
    call** — the full path is, precisely (all hops personally verified
    this pass): prompt text (item 29) names `"expand_node"` ->
    `crates/jcode-app-core/src/tool/communicate.rs:1959` (JSON Schema
    `action` enum literal) -> `communicate.rs:2629-2656` (the `"expand_node"
    =>` match arm, builds `Request::CommExpandNode`) ->
    `crates/jcode-app-core/src/server/comm_graph.rs:339-402`
    (`handle_comm_expand_node`, lifts plan to `TaskGraph` via
    `to_task_graph`, calls `dag::expand_node` at **line 368**, lowers back
    via `apply_task_graph`). Same shape for `complete_node`
    (`communicate.rs:2658-2684` -> `comm_graph.rs:409-...`, `dag::
    complete_node` at line 444) and `inject_gap`
    (`communicate.rs:2686-2718` -> `comm_graph.rs:482-...`, `dag::
    inject_from_gate` at line 511). TS equivalent: a typed RPC/tool-call
    dispatcher — a `Record<ActionName, (params) => Promise<Result>>` map
    (or a `switch` on a discriminated-union request type) whose handlers
    call into pure domain functions, structurally identical to this
    verified chain. Chapter 6, 2, 7 (this is the concrete mechanism behind
    "how does an agent's tool call become a real state mutation," relevant
    wherever the handbook discusses the swarm tool generally).

---

## 2. Grouping the table by "kind of pattern" (for Phase 3's page layout)

If Phase 3 wants the cheat sheet organized by *category* rather than strict
chapter order (probably clearer for a "glossary" chapter), these are natural
buckets pulled from the table above:

- **Fan-out/fan-in combinators**: items 5, 6, 7, 8.
- **Cancellation & lifecycle**: items 3, 9, 10, 11.
- **Shared state & singletons**: items 12, 13, 14.
- **Pub/sub & event delivery**: items 16, 17, 18, 7.
- **Indexing/lookup structures**: items 1, 15.
- **API/argument shape**: items 19, 27.
- **The DAG engine's specific architecture idioms**: items 21, 22, 23, 24,
  25, 26, 28.
- **The agent-facing protocol surface**: items 2, 29, 30.

---

## 3. Pointers into the design docs and crates (for "further reading")

Verified section locations (from the map's fully-read pass over both docs,
cross-checked by me against my own reads of the underlying code in Chapter
6's notes):

- **`docs/SWARM_ARCHITECTURE.md`** (318 lines) — the older, agent-first
  design doc. Header explicitly says it's "largely implemented" but
  superseded in framing by `SWARM_TASK_GRAPH.md`. Point readers who want
  the fuller **vocabulary and lifecycle picture** here: Roles (§Mode-gated
  spawning, lines 22-58), Agent Lifecycle States/Notifications (89-106),
  Completion Report Policy (108-124), Communication (194-254), Conflict
  Handling / No Locks (307-311, the exact source of the "optimistic by
  default" language Chapter 5 nuances). Flag for the reader, as the map
  does: the Agent Lifecycle States list (8 states) does **not** 1:1 match
  the code's `SwarmLifecycleStatus` enum (13 named variants + `Other`,
  `jcode-swarm-core/src/lib.rs:136-151`) — treat the doc as intent, the
  enum as current ground truth.
- **`docs/SWARM_TASK_GRAPH.md`** (605 lines) — the newer, DAG-first doc
  that Chapter 6 draws from most heavily. Point readers here for: §1a Deep
  vs Light comparison table (54-98, matches `dag::Mode` closely), §5
  Dataflow (195-216, matches `schedule::assemble_input`/
  `HandoffArtifact::render_section`), §6.4 "Implemented enforcement" (275-
  304, the single richest passage tying prose to real `DagError` variant
  names), §8a Communication rework / staged channel-deprecation migration
  (407-475, relevant to Chapter 7's "further reading"), §9 the worked
  example (478-549, Chapter 6's central narrative), §11 Suggested build
  order (593-605, historical/roadmap context only).
- **Crate-level pointers for going deeper than the handbook**:
  - `crates/jcode-plan/src/dag/` — the DAG engine itself: `mod.rs` (types),
    `ops.rs` (validated mutations), `schedule.rs` (ready-set/dispatch/
    dataflow hydration), `sim.rs` (deterministic simulator — **not opened
    by either Phase 1 or this pass**, flagged as a genuine "go read this
    yourself" pointer rather than a handbook citation), `tests.rs` (1392
    lines — larger than the engine itself; the best single source for
    every edge case the `DagError` variants exist to catch).
  - `crates/jcode-app-core/src/server/` — `swarm.rs` (member registry +
    plan-driven fan-out), `comm_session.rs` (spawn path),
    `comm_await.rs` (fan-in wait), `comm_graph.rs` (the DAG tool-call
    handlers), `state.rs` (`SwarmState`/`SwarmMember`).
  - `crates/jcode-app-core/src/tool/batch.rs` and `communicate.rs` — the
    intra-agent batch tool and the `swarm` tool's full action surface
    (~30 actions; the handbook only covers a handful).
  - `crates/jcode-base/src/bus.rs` — the global event bus and its full
    ~30-variant `BusEvent` vocabulary (394-466), most of which the handbook
    does not individually cover.
  - `crates/jcode-agent-runtime/src/lib.rs` — `InterruptSignal` and its
    five regression tests (142-283) for readers who want to see the
    lost-wakeup race pinned down directly.

---

## 4. Open questions / not independently verified this pass

1. Items drawn from Chapters 1-5, 7 in the table above rely on the Phase 1
   map's citations for files I did not personally re-open this pass beyond
   the handful marked "(re-verified)" above (`bus.rs`, `ChannelIndex`,
   `runtime.rs`'s `RuntimeTaskScope`). The map states these files were
   fully read in Phase 1 (`runtime.rs`, `comm_await.rs`, `batch.rs`,
   `agent-runtime/src/lib.rs`, `bus.rs`'s `BusEvent`/`Bus` all marked "OK,
   fully read" in the map's verification checklist) — I'm treating that as
   sufficient grounding for a glossary entry (a one-line pattern summary +
   citation), which is a lower bar than Chapter 6's requirement to quote
   full function bodies. If Phase 3/4 wants to quote a multi-line code
   block from any Chapter 1-5/7 file not marked "(re-verified)" above,
   re-open it first — this glossary's citations are verified to the
   "citation resolves, one-sentence claim is accurate" level, not to the
   "full function body re-read" level Chapter 6 required.
2. `dag/sim.rs` (155 lines) is intentionally listed as an unread
   further-reading pointer, not a citation — neither Phase 1 nor this pass
   opened it.
3. `jcode-protocol`'s exact wire-struct definitions for
   `Request::CommExpandNode` etc. (item 30) were not opened — the
   Rust-side call chain is fully verified, but the literal serialized JSON
   shape is not. Same caveat as noted in the Chapter 6 notes.
