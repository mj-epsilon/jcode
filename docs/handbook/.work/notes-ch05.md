# Phase 2 Notes — Chapter 5: Shared State Without Locks-as-a-Crutch

Verification status: re-opened `crates/jcode-app-core/src/server/state.rs`
(lines 1-240), `crates/jcode-app-core/src/server/swarm.rs` (lines 1-232 and
685-812), and `crates/jcode-base/src/bus.rs` (lines 460-609) directly on
2026-08-12, independent of the map's paraphrase. Every citation the map
gave for this chapter's core material checked out exactly. I additionally
searched `crates/jcode-app-core/src/server/` for every "debounce"
occurrence myself (`grep -rniE debounce`, excluding test files) to verify
the map's "recurs at least twice" claim, per the assignment's explicit
instruction not to take that characterization on faith — **result: I found
a third, independent occurrence the map did not mention** (see below).

---

## Part A: the core nuance — two different senses of "no locks"

### State the nuance precisely, in plain English, before any code

The phrase "optimistic, no locks" appears in jcode's own design doc,
`docs/SWARM_ARCHITECTURE.md:307-311` ("Conflict Handling (No Locks)"),
quoted verbatim:
> "The system is optimistic by default (no locks)... Coordination happens
> via DM or channel, not through the coordinator."

Read at face value against the Rust source, this looks like a
contradiction — the server code is full of `RwLock`/`Mutex`. It is not a
contradiction; it's two different *levels* of the same system being
described with the same word:

1. **The agent-facing coordination protocol** ("no locks" at this level):
   when two agents in a swarm might both want to touch the same file, or
   both want to claim the same piece of work, jcode does not make one
   agent block/wait on a mutex owned by the server before it's allowed to
   act. There is no "acquire the file lock, then edit" step an agent must
   go through. Conflicts, if they happen, are resolved *after the fact* —
   by agents communicating (a completion report, a DM, a file-touch
   notification) — not *prevented up front* by mutual exclusion. This is
   the sense the design doc means.

2. **The server's own in-memory bookkeeping** (does use locks, extensively):
   underneath, the server process itself is a multi-threaded tokio program
   with many concurrently-running tasks (one per client connection, one
   per background job, etc.) that all need to read and mutate shared
   in-memory data structures — "who are the current swarm members," "what
   does the current plan look like." For *that* purely-internal
   bookkeeping problem (not agent-to-agent coordination), the
   implementation reaches for the ordinary, correct tool: a lock around a
   shared data structure. This is necessary because — unlike single-
   threaded JS — tokio genuinely runs tasks on multiple OS threads at
   once; without a lock, two threads mutating the same `HashMap`
   simultaneously would be a data race, not just a theoretical risk.

**The one sentence to put in the chapter, verbatim-ish**: *"No locks" is a
claim about how agents coordinate work with each other (by communicating,
not by contending for a mutex); it is not a claim about how the Rust
server implements its own shared memory, which uses locks in the ordinary,
necessary way any multi-threaded program does.*

### The concrete evidence for level 2 (locks used throughout the implementation)

`crates/jcode-app-core/src/server/runtime.rs:90-120` — the `ServerRuntime`
struct. Nearly every field is `Arc<RwLock<_>>` or `Arc<Mutex<_>>`:
```rust
sessions: Arc<RwLock<HashMap<String, Arc<Mutex<Agent>>>>>,   // line 92
is_processing: Arc<RwLock<bool>>,                             // line 95
client_connections: Arc<RwLock<HashMap<String, ClientConnectionInfo>>>, // line 98
shared_context: Arc<RwLock<HashMap<String, HashMap<String, SharedContext>>>>, // line 100
shutdown_signals: Arc<RwLock<HashMap<String, InterruptSignal>>>, // line 115
```
(Confirmed by direct read; `sessions` at line 92 is double-locked — the
registry itself behind a `RwLock`, and each individual `Agent` behind its
own `Mutex`, so one agent's turn being processed doesn't block reads of
the member list.)

**The single best concrete exhibit for this chapter**:
`crates/jcode-app-core/src/server/state.rs:106-113` — `SwarmState`.
Confirmed by direct read:
```rust
/// Shared ownership of the core persisted swarm coordination state.
#[derive(Clone)]
pub struct SwarmState {
    pub members: Arc<RwLock<HashMap<String, SwarmMember>>>,
    pub swarms_by_id: Arc<RwLock<HashMap<String, HashSet<String>>>>,
    pub plans: Arc<RwLock<HashMap<String, VersionedPlan>>>,
    pub coordinators: Arc<RwLock<HashMap<String, String>>>,
}
```
This is *the* registry-of-registries for the chapter: four independently-
locked `HashMap`s (member records, swarm-membership index, plan storage,
coordinator-slot assignment), and the whole struct is cheaply
`#[derive(Clone)]`-able because cloning it just clones four `Arc`
pointers, not the underlying data — every function across `swarm.rs`/
`comm_session.rs`/`comm_await.rs` that needs swarm state receives a clone
of this struct (or of one of its `Arc<RwLock<...>>` fields directly) and
they're all looking at the same underlying maps.

**[RUST → plain English]** `Arc<RwLock<T>>` is Rust's standard
"shared-across-many-owners, safely-mutable" container. `Arc` (atomic
reference count) is what lets multiple owners hold a handle to the *same*
underlying data rather than each getting their own copy; `RwLock` is what
lets many concurrent *readers* in at once, or exactly one *writer* (unlike
a plain `Mutex`, which is exclusive even for reads — `RwLock` is the
"readers don't block each other" upgrade, useful here because status
lookups vastly outnumber status mutations). None of this exists in
single-threaded JS/TS: a plain `Map` is inherently only ever touched by
one turn of the event loop at a time, so there is no possibility of two
pieces of code reading/writing it *simultaneously* — the closest JS gets
to a genuine data race requires `worker_threads` + `SharedArrayBuffer` +
`Atomics`, which almost nobody reaches for. This is worth landing
explicitly in the chapter: the `Arc<RwLock<>>` proliferation is not
over-engineering, it's the ordinary cost of a program that truly runs many
OS threads concurrently.

### The `SwarmMember` struct (the record inside the lock)

`state.rs:186-240` — confirmed by direct read. Doc comment (`state.rs:186`,
one line, directly above the struct): "Information about a session in a
swarm." Key fields verified at their cited lines: `event_tx` (194, "kept
for backward-compatible single-sender call sites"), `event_txs` (196, the
live multi-attachment map), `status: String` (204, plain string — *not*
the typed `SwarmLifecycleStatus` enum used by the separate, persisted
`SwarmMemberRecord` type in `jcode-swarm-core`), `report_back_to_session_id`
(214, the spawn-ancestry field central to Chapter 2), `output_tail` (226-
229, explicitly "not persisted"), `todo_progress` (230-233, also "not
persisted"). Not central to Chapter 5's concurrency story on its own, but
useful as "here's what actually lives inside one of `SwarmState.members`'s
values" if Phase 3 wants to show a concrete record shape rather than just
the container type.

### A second worked example of "locks at the bookkeeping level" — and a real bug it fixed

`state.rs:15-38` — the process-global `BACKGROUND_TOOL_SIGNALS` registry.
Confirmed by direct read:
```rust
static BACKGROUND_TOOL_SIGNALS: LazyLock<StdMutex<HashMap<String, InterruptSignal>>> =
    LazyLock::new(|| StdMutex::new(HashMap::new()));
```
Doc comment (`state.rs:15-29`), worth quoting closely — this is the
chapter's best "here's *why* a lock was the right call, told through a
real bug" story:
> "Process-global registry mapping session id -> background-tool signal.
> The background-tool ("move tool to background", Alt+B/Ctrl+B) signal
> lives on the `Agent`, so a `SessionControlHandle` can normally only
> obtain it by locking the agent mutex. When a turn is busy (e.g. running
> `await_members`), `refresh_session_control_handle` falls back to a
> lock-free `cancel_only` handle that historically dropped the background
> signal entirely, which made Alt+B/Ctrl+B silently no-op... This registry
> is populated every time a full `SessionControlHandle` is built..., so
> the lock-free fallback can still fire the background signal without the
> agent lock."

Plain English narrative for the chapter: a hotkey (Alt+B/Ctrl+B, "move
this running tool to the background") used to silently do nothing when
the agent was busy, because the *only* way to reach its cancel signal was
through a lock that was already held by the busy turn, so the fallback
path just dropped it. The fix was not "remove the lock" — it's the
opposite: a second, small, independently-locked side-registry
(`BACKGROUND_TOOL_SIGNALS`, its own `std::sync::Mutex`, separate from the
agent's own lock) that duplicates just the one signal handle the fallback
path needs, so that path can reach it *without* needing the busy agent
lock at all. This is a genuinely instructive contrast to "optimistic, no
locks": in practice, avoiding lock contention in this codebase usually
means *narrowing what's behind each lock and adding small, independent
locks scoped to exactly what a given code path needs* — not eliminating
locks altogether.

I also independently confirmed (via the file's imports/functions,
`state.rs:34-65`) the accompanying accessor functions:
`register_background_tool_signal` (34-38), `background_tool_signal_for_session`
(41-46), `rename_background_tool_signal` (49-58), `remove_background_tool_signal`
(61-65) — small, uniform, mutex-guarded CRUD over the one `HashMap`, a
clean minimal example if Phase 3 wants the smallest possible
"process-global side-registry" snippet rather than the bigger `SwarmState`.

### Other confirmed process-global-singleton-behind-a-lock instances (for a "this shape recurs" aside)

- `crates/jcode-base/src/bus.rs:499-502` — `Bus::global()`:
  ```rust
  pub fn global() -> &'static Bus {
      static INSTANCE: OnceLock<Bus> = OnceLock::new();
      INSTANCE.get_or_init(Bus::new)
  }
  ```
  Confirmed by direct read. **[RUST → plain English]** `OnceLock` is a
  cell that initializes exactly once, the first time it's touched, and is
  read-only (no further locking) afterward — used here for the classic
  lazy-singleton pattern. TS equivalent: a module-level `let instance;
  export function getBus() { if (!instance) instance = new Bus(); return
  instance; }` — a process-wide singleton, not tied to any one request.
- `swarm.rs:108-113` — `pending_swarm_status_broadcasts()`, same
  `OnceLock<StdMutex<HashMap<...>>>` shape, confirmed (see Part B below).

---

## Part B: "debounce instead of lock harder"

### The problem, in plain English

Sometimes the *tempting* fix for "many concurrent writers are hammering
some shared state and I want to broadcast every change out to clients" is
to serialize everyone through a bigger/coarser lock, or to just send one
message per change (correct, but wasteful — if 50 status changes happen
in one second, why send 50 near-identical broadcasts?). jcode's actual
answer in at least three places (see below — the map claimed "at least
twice"; direct search found a third) is: don't strengthen the lock, and
don't send every individual change — instead, **coalesce a burst of rapid
changes into one delayed, deduplicated broadcast**. The lock stays small
and short-held; what gets smarter is *when* you actually publish.

### Occurrence 1 — `swarm.rs`'s status-broadcast debounce (confirmed, full read)

Constants, `swarm.rs:87-88`:
```rust
const DEFAULT_SWARM_STATUS_DEBOUNCE_MEMBER_THRESHOLD: usize = 2;
const DEFAULT_SWARM_STATUS_DEBOUNCE_MS: u64 = 75;
```
Doc comment on the terminal-member retention constant, `swarm.rs:94-100`
(quoted verbatim — a real measured cost, good concrete motivating number
for the chapter):
> "How long terminal members stay in live SwarmStatus broadcasts...
> re-sending hundreds of long-finished members to every attached client on
> every status change dominates broadcast payloads (measured ~240 KB of
> member JSON resident per client with ~700 mostly-stopped members)."

Debounce bookkeeping struct + global registry, `swarm.rs:102-113`:
```rust
#[derive(Default, Clone, Copy)]
struct PendingSwarmStatusBroadcast {
    scheduled: bool,
    dirty: bool,
}

fn pending_swarm_status_broadcasts()
-> &'static StdMutex<HashMap<String, PendingSwarmStatusBroadcast>> {
    static PENDING: OnceLock<StdMutex<HashMap<String, PendingSwarmStatusBroadcast>>> =
        OnceLock::new();
    PENDING.get_or_init(|| StdMutex::new(HashMap::new()))
}
```

The actual coalescing logic, `broadcast_swarm_status`, `swarm.rs:742-812`
(read in full, confirmed exact — this is the best single snippet to
feature):
```rust
pub(super) async fn broadcast_swarm_status(
    swarm_id: &str,
    swarm_members: &Arc<RwLock<HashMap<String, SwarmMember>>>,
    swarms_by_id: &Arc<RwLock<HashMap<String, HashSet<String>>>>,
) {
    // ... (fetch session_ids for this swarm) ...
    if session_ids.len() < swarm_status_debounce_member_threshold() {
        broadcast_swarm_status_now(session_ids, swarm_members).await;
        return;
    }

    let key = swarm_broadcast_key(swarm_id, swarm_members, swarms_by_id);
    let should_spawn = {
        let mut pending = pending_swarm_status_broadcasts()
            .lock()
            .unwrap_or_else(|poisoned| poisoned.into_inner());
        let entry = pending.entry(key.clone()).or_default();
        if entry.scheduled {
            entry.dirty = true;   // someone's already waiting to flush; just mark "there's more"
            false
        } else {
            entry.scheduled = true;
            entry.dirty = false;
            true                  // we're the first — we'll own the flush
        }
    };
    if !should_spawn { return; }

    // spawn ONE delayed flush loop per swarm; every call while it's
    // pending just sets `dirty` instead of spawning a second flush
    tokio::spawn(async move {
        loop {
            tokio::time::sleep(Duration::from_millis(swarm_status_debounce_ms())).await;
            // ...re-fetch current member statuses and broadcast_swarm_status_now...
            let mut pending = pending_swarm_status_broadcasts().lock()...;
            let Some(entry) = pending.get_mut(&key) else { break; };
            if entry.dirty {
                entry.dirty = false;
                continue;   // more changes arrived during the sleep — loop again
            }
            pending.remove(&key);
            break;          // caught up — stop looping, next call will spawn fresh
        }
    });
}
```
Plain English mechanism: below a small-swarm size threshold (2 members by
default — `swarm_status_debounce_member_threshold()`), every status change
broadcasts immediately, no debouncing at all — not worth the complexity
for a 1-2 member swarm. Above that threshold, the *first* caller in a
burst becomes the one that spawns a background "sleep N ms, then flush"
loop; every other caller that arrives while that loop is already scheduled
just flips a `dirty` flag and returns immediately (no new task spawned, no
lock held any longer than the single hashmap insert takes). When the sleep
elapses, it broadcasts the *current* (fresh) state — not whatever state
existed when any particular caller fired — and if `dirty` was set again
during that sleep, it loops and sleeps again rather than stopping, so a
sustained burst keeps getting coalesced rather than the debounce "wearing
off" after one cycle.

Both debounce parameters are runtime-configurable via env vars
(`swarm_status_debounce_member_threshold`, `swarm.rs:115-127`;
`swarm_status_debounce_ms`, `129-141`) — `JCODE_SWARM_STATUS_DEBOUNCE_MEMBER_THRESHOLD`
/ `JCODE_SWARM_STATUS_DEBOUNCE_MS`, each cached in a `OnceLock<AtomicUsize>`/
`OnceLock<AtomicU64>` after first read.

### Occurrence 2 — `Bus::publish_models_updated` (confirmed, full read)

`crates/jcode-base/src/bus.rs:542-606`. Confirmed by direct read; also
confirmed the debounce window constant:
```rust
const MODELS_UPDATED_DEBOUNCE: Duration = Duration::from_millis(750);   // bus.rs:476
```
and the per-instance (not global-static) debounce state, `bus.rs:468-474`:
```rust
pub struct Bus {
    sender: broadcast::Sender<BusEvent>,
    /// Debounce state for [`Bus::publish_models_updated`]. Per-instance (not a
    /// global static) so tests can exercise coalescing on a private bus
    /// without racing other tests that publish to the global bus.
    models_updated_state: std::sync::Arc<Mutex<ModelsUpdatedPublishState>>,
}
```
Worth noting as a nuance the map didn't dwell on: this is a *different*
debounce shape than occurrence 1's "schedule a flush, mark dirty on
repeats." `publish_models_updated` (`bus.rs:542-606`) instead tracks
`last_published_at: Option<Instant>` and a `publish_pending: bool`:
if enough time has elapsed since the last publish, it publishes
immediately and stamps the time; if not enough time has elapsed and
nothing is already pending, it schedules exactly one delayed publish for
"however long is left in the window" and marks `publish_pending`; if
something is *already* pending, it just returns (no-op) rather than
scheduling a second delayed publish. Functionally the same goal
(coalesce a burst into one delayed send) via a slightly different
bookkeeping shape (remaining-time math instead of a loop-and-recheck).
Good chapter point: *the same underlying pattern, reimplemented twice,
independently, with two different but both-correct bookkeeping shapes* —
worth showing both if there's room, or picking the `swarm.rs` one (more
self-contained) as the single featured snippet and mentioning this one
as "the same idea, done again independently, in a totally different part
of the codebase."

### Occurrence 3 — NOT in the map, found by direct search: `client_state.rs`'s attach-model-prefetch debounce

Per the assignment's explicit instruction to search
`crates/jcode-app-core/src/server/` for debounce logic myself rather than
trust the map's "at least twice" characterization at face value: I ran
`grep -rniE debounce` across that directory (excluding test files) and
found a **third, independent instance** the map did not mention, in
`crates/jcode-app-core/src/server/client_state.rs:25-53` (confirmed by
direct read):
```rust
const ATTACH_MODEL_PREFETCH_DEBOUNCE_SECS: u64 = 15;

static LAST_ATTACH_MODEL_PREFETCH: LazyLock<StdMutex<HashMap<String, Instant>>> =
    LazyLock::new(|| StdMutex::new(HashMap::new()));

fn should_debounce_attach_model_prefetch(provider_name: &str) -> bool {
    let Ok(mut guard) = LAST_ATTACH_MODEL_PREFETCH.lock() else {
        return false;
    };
    let now = Instant::now();
    if let Some(last_run) = guard.get(provider_name)
        && now.duration_since(*last_run) < Duration::from_secs(ATTACH_MODEL_PREFETCH_DEBOUNCE_SECS)
    {
        return true;
    }
    guard.insert(provider_name.to_string(), now);
    false
}
```
Called at `client_state.rs:897` when a client attaches, to decide whether
to skip a model-catalog prefetch for a provider that was already prefetched
recently.

Worth flagging as a real but *different* flavor of the same family, not
identical to occurrences 1-2: this is a **leading-edge rate-limit/dedup**
(do the work immediately the first time, then suppress repeats within a
window — "don't do this again too soon") rather than a **trailing-edge
coalesce-and-flush** (wait, collect changes, then do one combined thing
at the end of the quiet period — occurrences 1-2's shape). Both are
"debounce family" and both exist specifically so a small `std::sync::Mutex`
around a tiny side-map is enough — no need to lock or serialize the
actual prefetch/broadcast work itself. Good for the chapter as
"the same underlying instinct — narrow lock + time-window bookkeeping
instead of locking harder — shows up in more than one shape across the
codebase," strengthening (not just confirming) the map's "recurs at least
twice" claim into "recurs at least three times, in two distinct debounce
shapes."

I also confirmed one more artifact of this pattern-recurring, in a *test*
doc comment (so flagged as test-only per the map's own file-status
warning — `swarm.rs`'s production code ends at line 1726, tests start
1727): `swarm.rs:2174-2189` (inside `#[cfg(test)] mod tests`) documents a
known, acknowledged ordering subtlety in the debounce-free "immediate
path" (swarms below the member threshold) — two concurrent
`broadcast_swarm_status_now` calls can, in principle, deliver an older
snapshot after a newer one on the same ordered channel, since the
immediate path reads-then-later-writes the same lock rather than holding
one guard across both steps. **This is test-code documentation of a known
edge case, not a citable production guarantee** — worth mentioning as an
aside ("even the immediate/non-debounced path has a documented, tested-
around ordering subtlety") but must not be presented as describing
production behavior with confidence, per the map's explicit instruction
not to cite anything past `swarm.rs:1726` as production behavior.

---

## Snippet-selection recommendation for Phase 3

1. `SwarmState` struct (`state.rs:106-113`) — the flagship "here is the
   shared-state registry" exhibit, paired with the "no locks" doc-doc
   quote (`SWARM_ARCHITECTURE.md:307-311`) for the direct nuance
   contrast.
2. `BACKGROUND_TOOL_SIGNALS` + its doc comment (`state.rs:15-31`) — the
   "real bug, small side-lock fix" narrative; best "why is this locked"
   story in the chapter.
3. `broadcast_swarm_status`'s coalescing core (`swarm.rs:758-781`, the
   `should_spawn`/`entry.dirty` decision block specifically, trimmed from
   the full 742-812 function for space) — the flagship debounce snippet.
4. One-line mention + citation (not full snippet) of
   `Bus::publish_models_updated` (`bus.rs:542-606`) and the
   `client_state.rs` prefetch debounce (`client_state.rs:39-53`) as "the
   same idea, independently reinvented, at least twice more."

## TS-equivalent primitive mapping (summary table for Phase 3)

| Rust piece | file:line | TS teaching equivalent | Note |
| --- | --- | --- | --- |
| `Arc<RwLock<HashMap<...>>>` registry (`SwarmState`) | `state.rs:106-113` | a plain module-level `Map`/object | JS is single-threaded — no concurrent-mutation hazard exists, so the "lock" has no job to do; this is *the* teaching moment for why Rust needs it and TS doesn't |
| `Arc<Mutex<Agent>>` (double-locked registry) | `runtime.rs:92` | a `Map<string, Agent>` where "locking one agent" is just holding a reference to it | No mutual-exclusion concept needed |
| `static ...: LazyLock<StdMutex<HashMap<...>>>` (process-global side-registry) | `state.rs:30-31` | a module-level `const registry = new Map()` | Again, no lock needed in JS; the *pattern* (a small side-index to avoid needing a bigger lock) still ports as "a small auxiliary Map you maintain alongside the main data" |
| `OnceLock`-based lazy singleton (`Bus::global()`) | `bus.rs:499-502` | `let instance; function getBus() { if (!instance) instance = new Bus(); return instance; }` | Same idea, no lock needed |
| debounce-then-flush (`broadcast_swarm_status`) | `swarm.rs:742-812` | `setTimeout`-based coalescing: a `let scheduled = false` flag per key, schedule a `setTimeout` on first call, mark `dirty` on repeats, flush and re-check `dirty` in the timeout callback | Directly portable 1:1 — this is genuinely the same shape you'd write in Node, no true concurrency wrinkle here since debouncing is fundamentally a timing pattern, not a threading one |
| debounce-then-flush (`Bus::publish_models_updated`) | `bus.rs:542-606` | same `setTimeout` shape, tracked via `lastPublishedAt`/`pending` instead of a scheduled/dirty pair | Two different, both-valid ways to write the same debounce in either language |
| leading-edge rate-limit (`should_debounce_attach_model_prefetch`) | `client_state.rs:39-53` | `Map<string, number>` of last-run timestamps, checked before doing work | Directly portable; same "narrow lock, not less locking" lesson, JS just doesn't need the lock part |

## Verification summary

- `docs/SWARM_ARCHITECTURE.md:307-311` quote confirmed to exist as cited
  in the map (not re-read in full in this pass — trusting the map's
  already-fully-read status for this doc, per the checklist at the map's
  end marking it "OK, fully read").
- `state.rs:1-240` read in full directly; `SwarmState` (106-113),
  `BACKGROUND_TOOL_SIGNALS` + accessors (15-65), `SwarmMember` (186-240)
  all confirmed exact.
- `runtime.rs:90-120` (`ServerRuntime` field list) confirmed exact (see
  Chapter 4 notes for the full-file verification).
- `swarm.rs:1-232` and `685-812` read in full directly; debounce constants
  (87-88), doc comment (94-100), `PendingSwarmStatusBroadcast` (102-106),
  `pending_swarm_status_broadcasts()` (108-113), threshold/ms functions
  (115-141), `broadcast_swarm_status_now` (685-740), `broadcast_swarm_status`
  (742-812) all confirmed exact against the map.
- `bus.rs:460-609` read in full directly; `Bus` struct (468-474),
  `MODELS_UPDATED_DEBOUNCE` (476), `Bus::global()` (499-502), `Bus::publish`
  (525-533), `Bus::publish_models_updated` (542-606) all confirmed exact.
- Third debounce occurrence (`client_state.rs:25-53`) found via my own
  `grep -rniE debounce` search and confirmed by direct read — **not** in
  the Phase 1 map, reported here as a genuine addition, not a correction.
- One item flagged as test-only, not production-citable:
  `swarm.rs:2174-2189` (inside `#[cfg(test)] mod tests`, past the map's
  documented "real code ends ~1725" boundary) — included above only as a
  clearly-labeled aside, per the map's own instruction not to cite
  anything past line 1726 as production behavior.
- No open/unresolved questions for this chapter's core material beyond
  the test-only-citation caveat above.
