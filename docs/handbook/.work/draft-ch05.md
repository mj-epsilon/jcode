# Chapter 5 — Shared State Without Locks-as-a-Crutch

jcode's own design documentation makes a bold claim about how agents in a
swarm coordinate:

> "The system is optimistic by default (no locks)... Coordination happens
> via DM or channel, not through the coordinator."
> — `docs/SWARM_ARCHITECTURE.md:307-311`

Then you open the Rust source and find `RwLock` and `Mutex` wrapped around
almost every shared data structure in the server. Is the design doc lying?

No — and untangling *why not* is one of the more useful things a
non-Rust reader can take from this codebase, because the confusion is
common and the resolution is simple once you see it: **"no locks" and
"uses locks" are true statements about two different layers of the same
system.**

## Two senses of "no locks"

**Layer 1 — how agents coordinate with each other.** When two agents in a
swarm might both want to touch the same file, or claim the same piece of
work, jcode does not make one agent block on a mutex owned by the server
before it's allowed to act. There's no "acquire the file lock, then edit"
step an agent has to go through. If a conflict happens, it gets resolved
*after the fact* — through agents communicating (a completion report, a
direct message, a file-touch notification) — not *prevented up front* by
mutual exclusion. This is the layer the design doc is describing, and at
this layer the claim is accurate: there is no lock an agent waits on to
act.

**Layer 2 — how the server keeps its own bookkeeping consistent.**
Underneath all of that, the server itself is an ordinary multi-threaded
program. It has one task running per client connection, one per
background job, and so on — all genuinely running at once, on different
OS threads, all needing to read and write shared in-memory data like "who
are the current swarm members" or "what does the current plan look like."
For that purely internal problem — not agent-to-agent coordination, just
keeping a shared `HashMap` from being corrupted by two threads writing to
it simultaneously — the implementation reaches for the ordinary, correct
tool: a lock around the shared structure. This isn't optional the way
Layer 1's locklessness is a design choice; it's the baseline requirement
of any correct multi-threaded program.

The sentence worth keeping in your head for the rest of this chapter:
*"no locks" is a claim about how agents coordinate work with each other,
not a claim about how the server implements its own shared memory.*

### Layer 2, made concrete

The clearest example is `SwarmState` — the registry-of-registries that
most of the server's swarm logic reads and writes:

```rust
// jcode: crates/jcode-app-core/src/server/state.rs:106-113
/// Shared ownership of the core persisted swarm coordination state.
#[derive(Clone)]
pub struct SwarmState {
    pub members: Arc<RwLock<HashMap<String, SwarmMember>>>,
    pub swarms_by_id: Arc<RwLock<HashMap<String, HashSet<String>>>>,
    pub plans: Arc<RwLock<HashMap<String, VersionedPlan>>>,
    pub coordinators: Arc<RwLock<HashMap<String, String>>>,
}
```

Four independently-locked hash maps: member records, a swarm-to-members
index, plan storage, and coordinator assignments. `Arc` (atomic reference
count) lets many owners hold a handle to the *same* underlying data rather
than each getting a private copy — cloning `SwarmState` just clones four
pointers, not the maps behind them. `RwLock` lets many concurrent
*readers* in at once, or exactly one *writer* — unlike a plain mutex,
which would make even simultaneous readers wait on each other. That
distinction matters here specifically because status lookups vastly
outnumber status changes.

None of this machinery has a reason to exist in single-threaded JS. A
plain `Map` is only ever touched by one turn of the event loop at a time
— there's no possibility of two pieces of code reading and writing it
*simultaneously*, so there's nothing for a lock to protect against. (The
one corner of JS where this stops being true is `worker_threads` combined
with `SharedArrayBuffer` and `Atomics` — real shared memory across real
OS threads — which almost nobody reaches for.) The `Arc<RwLock<...>>`
you see all over jcode's server code isn't over-engineering. It's the
ordinary, unavoidable cost of a program that truly runs many threads at
once.

```typescript
// idiomatic TS equivalent — this code does not exist in jcode
// No lock needed: within one turn of Node's event loop, nothing else
// can be mid-write to this map while your code runs.
class SwarmState {
  members = new Map<string, SwarmMember>();
  swarmsById = new Map<string, Set<string>>();
  plans = new Map<string, VersionedPlan>();
  coordinators = new Map<string, string>();
}
```

### A real bug that a *narrower* lock fixed

It would be easy to conclude "locks are annoying, minimize them." jcode's
own history argues for something more specific: when a lock gets in the
way, the fix is usually a smaller, more targeted lock — not the absence of
one. `state.rs` carries a small process-global registry that exists
because of exactly this lesson:

```rust
// jcode: crates/jcode-app-core/src/server/state.rs:15-31
/// Process-global registry mapping session id -> background-tool signal.
///
/// The background-tool ("move tool to background", Alt+B/Ctrl+B) signal lives on
/// the `Agent`, so a `SessionControlHandle` can normally only obtain it by
/// locking the agent mutex. When a turn is busy (e.g. running `await_members`),
/// `refresh_session_control_handle` falls back to a lock-free `cancel_only`
/// handle that historically dropped the background signal entirely, which made
/// Alt+B/Ctrl+B silently no-op (`BACKGROUND_TOOL_SIGNAL_FIRE result=no_signal_handle`).
///
/// This registry is populated every time a full `SessionControlHandle` is built
/// (which always has both the session id and the correct signal), so the
/// lock-free fallback can still fire the background signal without the agent
/// lock. Entries are keyed by session id; renames/removals reuse
/// [`rename_background_tool_signal`]/[`remove_background_tool_signal`] alongside
/// the existing shutdown-signal lifecycle.
static BACKGROUND_TOOL_SIGNALS: LazyLock<StdMutex<HashMap<String, InterruptSignal>>> =
    LazyLock::new(|| StdMutex::new(HashMap::new()));
```

The story in plain English: a hotkey ("move this running tool to the
background") used to silently do nothing while an agent was busy. The
*only* way to reach the signal it needed to fire was through the agent's
own lock — and that lock was already held by the busy turn, so the
fallback path just gave up and dropped the request. The fix wasn't to
remove the lock. It was the opposite: add a second, small, independently
locked side-registry that duplicates just the one signal handle the
fallback path actually needs, so that path can reach it without ever
touching the busy agent's lock.

This is the practical version of "optimistic, no locks" you'll actually
find in the implementation: not zero locks, but *narrowing what sits
behind each lock* so no single lock becomes a bottleneck other code paths
get stuck behind.

```typescript
// idiomatic TS equivalent — this code does not exist in jcode
// No lock needed in JS, but the *pattern* still ports: a small auxiliary
// side-index kept next to the main data, so a code path that can't touch
// the main structure right now can still reach the one thing it needs.
const backgroundToolSignals = new Map<string, InterruptSignal>();

function registerBackgroundToolSignal(sessionId: string, signal: InterruptSignal) {
  backgroundToolSignals.set(sessionId, signal);
}

function backgroundToolSignalForSession(sessionId: string): InterruptSignal | undefined {
  return backgroundToolSignals.get(sessionId);
}
```

### The same shape again: lazy singletons

One more recurring pattern worth naming before moving to debouncing: a
process-wide singleton built with a one-time-initializing cell.

```rust
// jcode: crates/jcode-base/src/bus.rs:499-502
pub fn global() -> &'static Bus {
    static INSTANCE: OnceLock<Bus> = OnceLock::new();
    INSTANCE.get_or_init(Bus::new)
}
```

`OnceLock` initializes exactly once, on first use, and is read-only after
that — no ongoing locking cost. The same shape shows up again for the
debounce bookkeeping later in this chapter
(`swarm.rs:108-113`).

```typescript
// idiomatic TS equivalent — this code does not exist in jcode
let instance: Bus | undefined;
function getBus(): Bus {
  if (!instance) instance = new Bus();
  return instance;
}
```

## Part B: debounce instead of locking harder

There's a second instinct worth its own section, because it recurs
independently across the codebase in a way that's genuinely interesting:
when many concurrent writers are hammering some shared state and you want
to tell clients about every change, the *tempting* fix is either a bigger
lock (serialize everyone) or a message per change (correct, but wasteful —
fifty status changes in one second doesn't need fifty near-identical
broadcasts). jcode's actual answer, found independently at least three
times in the server codebase, is neither: keep the lock small and
short-held, and instead **coalesce a burst of rapid changes into one
delayed, deduplicated broadcast.**

### Occurrence 1 — swarm status broadcasts

The concrete cost that motivates this is stated directly in the source:

> "How long terminal members stay in live SwarmStatus broadcasts...
> re-sending hundreds of long-finished members to every attached client on
> every status change dominates broadcast payloads (measured ~240 KB of
> member JSON resident per client with ~700 mostly-stopped members)."
> — `crates/jcode-app-core/src/server/swarm.rs:94-100`

The debounce bookkeeping is a tiny struct behind its own small lock:

```rust
// jcode: crates/jcode-app-core/src/server/swarm.rs:87-88, 102-113
const DEFAULT_SWARM_STATUS_DEBOUNCE_MEMBER_THRESHOLD: usize = 2;
const DEFAULT_SWARM_STATUS_DEBOUNCE_MS: u64 = 75;

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

And the coalescing logic itself:

```rust
// jcode: crates/jcode-app-core/src/server/swarm.rs:758-781 (trimmed from the
// full function at 742-812)
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
        entry.dirty = true; // someone's already waiting to flush; just mark "there's more"
        false
    } else {
        entry.scheduled = true;
        entry.dirty = false;
        true // we're first — we own the flush
    }
};
if !should_spawn {
    return;
}

tokio::spawn(async move {
    loop {
        tokio::time::sleep(Duration::from_millis(swarm_status_debounce_ms())).await;
        // ...re-fetch current member statuses and broadcast the fresh snapshot...
        let mut pending = pending_swarm_status_broadcasts().lock()...;
        let Some(entry) = pending.get_mut(&key) else { break; };
        if entry.dirty {
            entry.dirty = false;
            continue; // more changes arrived during the sleep — loop again
        }
        pending.remove(&key);
        break; // caught up — stop; the next call spawns fresh
    }
});
```

The mechanism, in plain English: below a small threshold (2 members by
default), every status change broadcasts immediately — not worth
debouncing a 1-2 member swarm. Above that threshold, the *first* caller in
a burst becomes the one that spawns a "sleep, then flush" loop. Every
other caller that arrives while that loop is already scheduled just flips
a `dirty` flag and returns — no new task, no lock held longer than a
single hashmap insert. When the sleep elapses, the loop broadcasts the
*current* state, not whatever state existed when any particular caller
fired, and if `dirty` got set again during the sleep, it loops and sleeps
again rather than stopping — so a sustained burst keeps getting coalesced
instead of the debounce "wearing off" after one cycle.

```typescript
// idiomatic TS equivalent — this code does not exist in jcode
type PendingEntry = { timer: ReturnType<typeof setTimeout>; dirty: boolean };
const pending = new Map<string, PendingEntry>();

function broadcastSwarmStatus(swarmId: string, members: Map<string, SwarmMember>) {
  const sessionIds = getSessionIdsFor(swarmId);
  if (sessionIds.length < DEBOUNCE_MEMBER_THRESHOLD) {
    broadcastNow(sessionIds, members);
    return;
  }

  const entry = pending.get(swarmId);
  if (entry) {
    entry.dirty = true; // already scheduled — just note there's more
    return;
  }

  const timer = setTimeout(function flush() {
    broadcastNow(getSessionIdsFor(swarmId), members);
    const current = pending.get(swarmId)!;
    if (current.dirty) {
      current.dirty = false;
      current.timer = setTimeout(flush, DEBOUNCE_MS);
    } else {
      pending.delete(swarmId);
    }
  }, DEBOUNCE_MS);

  pending.set(swarmId, { timer, dirty: false });
}
```

This one ports almost 1:1. Debouncing is fundamentally a timing pattern,
not a threading one, so it's one of the few places in this chapter where
the Rust and the idiomatic TS really do look like the same code in two
languages.

### Occurrence 2 — same idea, different bookkeeping shape

`Bus::publish_models_updated` (`crates/jcode-base/src/bus.rs:542-606`)
solves the identical problem — coalesce a burst of "models updated"
events into one delayed publish — but tracks it differently: instead of a
`scheduled`/`dirty` pair, it keeps `last_published_at: Option<Instant>`
and a `publish_pending: bool`. If enough time has passed since the last
publish, it publishes immediately; if not, and nothing is already
pending, it schedules exactly one delayed publish for whatever time
remains in the window; if something is already pending, it's a no-op. Same
goal, same family, independently reinvented with different bookkeeping —
worth knowing this exists, not worth a second full code block.

### Occurrence 3 — a different flavor: leading-edge rate limiting

The third instance wasn't in this chapter's original research map — it
turned up from directly grepping the server crate for "debounce," which is
worth mentioning because it's a good reminder that "the notes said two"
is worth checking rather than trusting outright:

```rust
// jcode: crates/jcode-app-core/src/server/client_state.rs:25-53 (called at line 897)
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

This one is a genuinely different shape, worth distinguishing by name: the
first two occurrences are **trailing-edge coalescing** — wait, collect
changes, do one combined thing at the end of a quiet period. This one is
**leading-edge rate limiting** — do the work immediately the first time,
then suppress repeats within a cooldown window ("don't do this again too
soon," here used to skip a model-catalog prefetch for a provider that was
already prefetched recently on client attach). Different shape, same
underlying instinct: a small lock around a tiny side-map is enough,
because what's being protected is a few bytes of bookkeeping, not the
actual work.

```typescript
// idiomatic TS equivalent — this code does not exist in jcode
const lastAttachModelPrefetch = new Map<string, number>();
const DEBOUNCE_MS = 15_000;

function shouldDebounceAttachModelPrefetch(providerName: string): boolean {
  const now = Date.now();
  const lastRun = lastAttachModelPrefetch.get(providerName);
  if (lastRun !== undefined && now - lastRun < DEBOUNCE_MS) {
    return true;
  }
  lastAttachModelPrefetch.set(providerName, now);
  return false;
}
```

## Takeaways

- "No locks" in jcode's design docs describes how agents coordinate with
  each other (by communicating, not by contending for a mutex) — not how
  the Rust server manages its own in-memory state, which uses ordinary
  locks because it's an ordinary multi-threaded program.
- None of the lock machinery you see in the Rust source has a reason to
  exist in single-threaded JS — a plain `Map` does the same job, because
  there's no possibility of two pieces of code touching it at the same
  instant.
- When a lock in this codebase caused a real problem, the fix was a
  smaller, more targeted lock around exactly what a blocked code path
  needed — not the removal of locking altogether.
- The debounce pattern shows up at least three times, independently, in
  two distinct shapes: trailing-edge coalescing (wait, collect, flush
  once) and leading-edge rate limiting (do it now, suppress repeats). Both
  exist so the lock stays small; what gets smarter is *when* you act, not
  how hard you serialize access.
