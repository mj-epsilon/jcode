# Chapter 7 — Communication Topology

If Chapter 3 was about running things in parallel, this chapter is about
what agents actually say to each other once they're running: who can send a
message to whom, who can read what about another agent's state, and how a
piece of news travels from one part of the system to every place that cares
about it.

One framing note before the code, because it changes how you should read
everything below. jcode has two design docs describing this territory, and
they're not equally current. `docs/SWARM_ARCHITECTURE.md` opens by
describing itself as "largely implemented," but points to
`docs/SWARM_TASK_GRAPH.md` for the newer, DAG-first model that supersedes
its agent-first framing — and says its own communication section is
mid-migration. `SWARM_TASK_GRAPH.md` §8a lays out a four-step plan to trim
a chattier, chat-style communication surface down to something leaner:

1. **Done.** Completion artifacts flow to dependents automatically.
2. **Done.** Broadcasts are scoped to the sender's spawned subtree, not the
   whole swarm.
3. Migrate existing flows off channels and shared-context. (pending)
4. Deprecate, then remove, the redundant chat primitives. (pending)

So: what follows is "what's actually live today," described mostly in the
older doc's vocabulary (DMs / broadcast / channels), with step 2 of that
migration — subtree-scoped broadcast — already landed in code. Treat this as
a system partway through simplifying itself, not a finished design.

---

## DMs, subtree broadcast, and channels

All three routing shapes go through one function:
`handle_comm_message` in
`crates/jcode-app-core/src/server/client_comm_message.rs`. A single call
picks its delivery shape based on which optional fields the caller filled
in:

```rust
// jcode: crates/jcode-app-core/src/server/client_comm_message.rs:217-223
let scope = if resolved_to_session.is_some() {
    "dm"
} else if channel.is_some() {
    "channel"
} else {
    "broadcast"
};
```

`to_session` wins if present, then `channel`, and if neither is set it
falls through to broadcast. This single function auto-routing among three
delivery shapes is exactly the property `SWARM_TASK_GRAPH.md` §8a points to
as evidence the surface has too many overlapping primitives — worth reading
as the design doc's own retrospective critique of the code you're looking
at, not an endorsement of it.

**DMs.** A DM target can be an exact session ID, or a swarm-unique
"friendly name" resolved by `resolve_dm_target_session`
(`client_comm_message.rs:35-80`). If a friendly name matches more than one
session, that's a hard error naming every match — it doesn't silently pick
one for you.

**Subtree broadcast.** This is the concrete implementation of migration
step 2 above:

```rust
// jcode: crates/jcode-app-core/src/server/client_comm_message.rs:230-249
// Broadcast-style sends are subtree-scoped: a sender reaches only the
// agents it (transitively) spawned, via the report-back ancestry chain.
// The swarm coordinator keeps whole-swarm reach as an escape hatch.
// This prevents one agent from producing a member-cap-sized
// notification storm (see docs/SWARM_TASK_GRAPH.md section 8a).
let subtree_broadcast_targets: Vec<String> = {
    let members = swarm_members.read().await;
    let sender_is_coordinator = members
        .get(&from_session)
        .is_some_and(|member| member.role == "coordinator");
    swarm_session_ids
        .iter()
        .filter(|session_id| *session_id != &from_session)
        .filter(|session_id| {
            sender_is_coordinator
                || super::swarm_is_self_or_ancestor(&members, &from_session, session_id)
        })
        .cloned()
        .collect()
};
```

A plain agent's "broadcast" only reaches sessions it transitively spawned —
tracked via the report-back ancestry chain (Chapter 2's territory). Only
the coordinator gets true whole-swarm reach, as a deliberate escape hatch.
Notice the reused primitive: `swarm_is_self_or_ancestor` is the same
authorization check used elsewhere in `swarm.rs` to decide who may stop a
session — one function, two different jobs.

Also notice `member.role == "coordinator"` here compares against a bare
string, not a typed enum. Hold that thought — it's the seed of the biggest
finding in this chapter, in the last section below.

**Channels.** The least emphasized of the three, and the code shows it:

```rust
// jcode: crates/jcode-app-core/src/server/client_comm_message.rs:253-269
} else if let Some(ref channel_name) = channel {
    let subs = channel_subscriptions.read().await;
    let index = ChannelIndex {
        by_swarm_channel: subs.clone(),
        by_session: HashMap::new(),
    };
    let channel_members = index.members(&swarm_id, channel_name);
    if channel_members.is_empty() {
        // No subscribers: fall back to the subtree scope rather than
        // blasting the whole swarm.
        subtree_broadcast_targets.clone()
    } else {
        channel_members
            .into_iter()
            .filter(|session_id| session_id != &from_session)
            .collect()
    }
}
```

An empty channel silently falls back to the subtree-broadcast scope rather
than erroring or delivering nothing — channels behave here as "a broadcast
pre-filter that no-ops back to broadcast if unused." That's consistent with
`SWARM_ARCHITECTURE.md`'s own framing of channels as discouraged in favor
of DMs. It's also worth noting the code builds a brand-new `ChannelIndex`
from the raw subscription map on every single call, rather than holding a
persistent index — the actual long-lived state is just the underlying
`channel_subscriptions` map; `ChannelIndex` is a one-shot lookup helper
around it, not a registry in its own right.

Whichever of the three shapes resolves a target list, delivery itself goes
through one shared helper, `fanout_session_event`, which per
`SWARM_ARCHITECTURE.md` queues these as "soft interrupts" injected into a
running agent at safe points. That specific injection-timing claim comes
from the design doc — this chapter's source notes did not independently
trace `queue_soft_interrupt_for_session`'s implementation, so treat "safe
points" as the documented intent rather than something verified line by
line here.

---

## Three ways to read another agent's state

A swarm member can ask about another member in three different ways, each
with a different cost and a different availability guarantee. All three
live in `crates/jcode-app-core/src/server/comm_sync.rs`, and all three sit
behind one shared gate: `ensure_same_swarm_access`
(`comm_sync.rs:144-176`) — requester and target must share a `swarm_id`
before any of what follows runs.

**Status snapshot — `handle_comm_status`, `comm_sync.rs:249-324`.** The
cheapest tier, and designed to never block. The core snapshot is assembled
entirely from the `swarm_members` registry — a data-structure lock
(`Arc<RwLock<HashMap<...>>>`), not the per-agent mutex — plus file-touch and
connection state. It only *opportunistically* reaches for the target
agent's own lock, using `try_lock()`, to grab provider/model name; if that
fails because the agent is busy, those two fields just come back empty
rather than the call blocking or erroring.

**Summary read — `handle_comm_summary`, `comm_sync.rs:194-243`.** A step up
in cost: it must lock the target agent (`try_lock`) to call
`agent.get_tool_call_summaries(limit)` (default limit 10) — a short
activity feed of recent tool calls. Unlike status, this tier does *not*
degrade quietly if the lock fails; it returns an explicit error, `"Session
'{}' is busy; try summary again shortly"`, with a `retry_after_secs: 1`
hint. Bounded and lightweight when it works, but it can fail transiently
under load.

**Full context read — `handle_comm_read_context`, `comm_sync.rs:326-382`.**
The expensive, gated tier: the entire message history via
`agent.get_history()`. Two layers of protection beyond the other two tiers.
First, `can_read_full_context` (`comm_sync.rs:178-192`) restricts this to
the session itself or a session whose `role == "coordinator"` — again a
bare-string comparison, not the typed role enum — with an explicit error
for anyone else: "Only the coordinator, worktree manager, or the target
session may read full context. Use summary for lightweight access." Second,
the same try-lock-or-explicit-busy-error pattern as the summary tier.

Same underlying data — an `Agent` behind a mutex, plus a lock-free metadata
registry — exposed at three deliberately different cost points:
always-available-but-shallow, cheap-but-can-fail-busy, and
expensive-and-permission-gated. It's a clean design to hold onto as a
pattern in its own right: when multiple callers need different amounts of
detail about the same live object, give them different endpoints with
different locking guarantees, rather than one endpoint that's either always
slow or always risks blocking on a busy resource.

---

## The global event bus

Separate from any of the routing above, jcode has one process-wide event
bus that everything else in the UI listens to:
`crates/jcode-base/src/bus.rs`.

```rust
// jcode: crates/jcode-base/src/bus.rs:499-502
pub fn global() -> &'static Bus {
    static INSTANCE: OnceLock<Bus> = OnceLock::new();
    INSTANCE.get_or_init(Bus::new)
}
```

A lazily-initialized, process-wide singleton. Structurally, `Bus` is just a
`broadcast::Sender<BusEvent>` (plus some private debounce bookkeeping) —
initialized with a fixed 256-slot ring buffer
(`broadcast::channel(256)`). `BusEvent` is a roughly-30-variant enum;
the ones you'll recognize from Chapter 3 are `BatchProgress` (the batch
tool's incremental progress), `FileTouch`, `SwarmOutputTail`, and
`SwarmAwaitCompleted` — the exact event `finalize_await` publishes when a
background wait completes, which is the hand-off point between Chapter 3's
Pattern 3 and this bus.

Publishing is deliberately unconcerned with whether anyone's listening:

```rust
// jcode: crates/jcode-base/src/bus.rs:525-533 (publish, abbreviated)
let _ = self.sender.send(event);
```

The send's `Result` is thrown away — zero receivers isn't a failure for a
fire-and-publish bus. There's one carve-out: `UpdateStatus` events are also
cached in a separate mutex, specifically so a subscriber that connects
*after* the last publish can still ask for the latest value
(`Bus::latest_update_status()`) instead of missing it entirely — a
documented workaround for the fact that a broadcast channel has no general
replay/history mechanism.

That last point matters for Chapter 3 readers: the fixed 256-slot buffer is
exactly why `comm_await.rs`'s handling of `RecvError::Lagged` matters in
practice — any subscriber that falls more than 256 unconsumed events behind
starts missing events, and each subscriber lags independently (a slow
subscriber doesn't block the publisher or anyone else).

Worth naming explicitly, because it's a real topology distinction: the
global `Bus` is *true* broadcast — one `tokio::sync::broadcast` channel,
every subscriber gets every message. Swarm status/plan fan-out
(`broadcast_swarm_status`, `broadcast_swarm_plan_with_previous`,
`swarm.rs:685-970`) looks like broadcast from a distance — many recipients
get the same update — but under the hood it's a loop over each member's own
individual `mpsc` sender:

```
// looks like: fanout_session_event(swarm_members, &sid, event.clone())
// called once per affected session id, not one send on a shared channel
```

So: one process has (at least) two different things that both produce a
"many people got the same message" effect, but they're built on physically
different primitives — a single shared broadcast channel vs. a fan-out loop
over point-to-point channels. The TS analogy: the global bus is one
`EventEmitter` topic everyone subscribes to; the swarm status/plan fan-out
is a `Map<sessionId, sendFn>` you iterate and call individually.

### The TypeScript equivalent

```typescript
// idiomatic TS equivalent — this code does not exist in jcode
type BusEvent =
  | { type: "BatchProgress"; completed: number; total: number }
  | { type: "SwarmAwaitCompleted"; swarmId: string }
  | { type: "UpdateStatus"; status: string }; // ...and more variants

class Bus extends EventEmitter {
  private static instance: Bus;
  private lastUpdateStatus: BusEvent | undefined;

  static global(): Bus {
    return (Bus.instance ??= new Bus());
  }

  publish(event: BusEvent): void {
    if (event.type === "UpdateStatus") {
      this.lastUpdateStatus = event; // replay workaround, same as jcode
    }
    this.emit("event", event); // no error if nobody's listening
  }

  latestUpdateStatus(): BusEvent | undefined {
    return this.lastUpdateStatus;
  }
}
```

Node's `EventEmitter` doesn't drop events under backpressure the way a
bounded `broadcast` channel does — it just keeps emitting synchronously to
whoever's attached. The translation above keeps the API shape (global
singleton, fire-and-forget publish, one cached "last value" escape hatch for
late subscribers) without claiming the same backpressure behavior, since
that would be dishonest about how `EventEmitter` actually works.

---

## The interesting finding: `SwarmRole`/`SwarmLifecycleStatus` and the rewrite-on-reload

Here's a genuinely worthwhile finding, not just a code tour. You may have
noticed above that all the live routing and permission checks compare
`role`/`status` as bare strings — `member.role == "coordinator"`,
`member.status == "running"`. That's because, in memory, they *are* plain
strings:

```rust
// jcode: crates/jcode-app-core/src/server/state.rs:204,218 (SwarmMember fields)
status: String, // "Lifecycle status (ready, running, completed, failed, stopped, etc.)"
role: String,   // "Role: \"agent\" or \"coordinator\""
```

But there's a second, typed representation used at the persistence
boundary — `SwarmMemberRecord`, in `crates/jcode-swarm-core/src/lib.rs`,
with `status: SwarmLifecycleStatus` (13 named variants plus an `Other(String)`
catch-all) and `role: SwarmRole` (`Agent`/`Coordinator`/`Other(String)`).
Its doc comment calls it "the durable, persistable portion of a swarm
member."

So there are two vocabularies. Does anything actually convert between
them, or do they just coexist? Conversion code exists, and it turns out to
do more than a naive round-trip.

**Saving:** `SwarmMember::durable_record`, `state.rs:243-258`:

```rust
// jcode: crates/jcode-app-core/src/server/state.rs:243-258
pub fn durable_record(&self) -> SwarmMemberRecord {
    SwarmMemberRecord {
        session_id: self.session_id.clone(),
        working_dir: self.working_dir.clone(),
        swarm_id: self.swarm_id.clone(),
        swarm_enabled: self.swarm_enabled,
        status: SwarmLifecycleStatus::from(self.status.clone()),
        detail: self.detail.clone(),
        task_label: self.task_label.clone(),
        friendly_name: self.friendly_name.clone(),
        report_back_to_session_id: self.report_back_to_session_id.clone(),
        latest_completion_report: self.latest_completion_report.clone(),
        role: SwarmRole::from(self.role.clone()),
        is_headless: self.is_headless,
    }
}
```

`SwarmLifecycleStatus::from(String)` and `SwarmRole::from(String)`
(`jcode-swarm-core/src/lib.rs:174-193` and `:107-115`) match the string
against every known literal — `"ready" => Self::Ready`, `"running" =>
Self::Running`, and so on — and fall through to `Self::Other(value)` for
anything unrecognized. The conversion is total (it never panics) and
open-world (an unrecognized string is preserved rather than dropped), but
that also means a typo in a status string silently becomes `Other("...")`
rather than getting caught as a bug.

The reverse direction, loading, calls `.as_str()` on each enum to get back
a canonical lowercase string. For any string that maps to a *named*
variant, the round trip is lossless — but only if the input matched the
canonical lowercase form to begin with. `"Ready"` (capital R) would land in
`Other("Ready")` on the way in, not `Ready` — a real, if narrow, footgun
that the typed round-trip doesn't fully protect against.

That much would already be a reasonable chapter finding: a hand-written,
total, open-world string↔enum conversion, called on every swarm-state
save/load cycle (confirmed via `swarm_persistence.rs`'s
`to_persisted_member`/`from_persisted_member`, which call
`durable_record`/`from_record` respectively). But there's a sharper detail
underneath it.

**The round trip is not actually an identity function in practice**,
because of what happens on load, in between reading the persisted record
and reconstructing the live `SwarmMember`:

```rust
// jcode: crates/jcode-app-core/src/server/swarm_persistence.rs:341-390
// (recover_member_status, paraphrased structure — see file for exact match arms)
```

`recover_member_status` deliberately rewrites certain statuses as they come
back off disk, because a live process context — the running task, the open
event channel — cannot survive a server restart even though the *status
string* that was written to disk can:

- A persisted `Running` status becomes `Crashed` on load — nothing can
  genuinely still be "running" after the server that was running it
  restarted.
- A persisted `Ready` status becomes `Stopped` on load. The code's own
  comment explains the bug this fixes: without this rewrite, every reload
  would resurrect hundreds of detached historical clients as "ready" again.
- A headless member sitting in any non-terminal status becomes `Crashed`
  too, with an explicit carve-out for statuses that are legitimately final
  (`Completed`, `Done`, `Failed`, `Stopped` are left alone).

So the answer to "does a translation layer exist between the two
vocabularies" isn't just yes — it's that the translation layer is *where
the codebase enforces that a persisted status can never lie about having
survived a restart*. That's the enum's real behavioral job in this system,
beyond serialization: `recover_member_status` pattern-matches on it, and
because it's a compiler-checked enum rather than string literals, that
match is exhaustive — you can't add a new `SwarmLifecycleStatus` variant
and forget to decide what `recover_member_status` should do with it without
the compiler telling you.

Read as a design lesson, this is a nice concrete instance of "put the type
system where correctness actually matters, not uniformly everywhere." Live,
in-process code compares against string literals freely, because those
comparisons are cheap and the strings never leave the process boundary in a
way that risks silent corruption. The one place that *does* risk something
expensive and hard to notice — silently believing a session is still
running when the process that ran it is gone — is exactly where the
codebase opts into a typed, exhaustively-matched enum instead.

### The TypeScript equivalent

```typescript
// idiomatic TS equivalent — this code does not exist in jcode
type SwarmLifecycleStatus =
  | "ready" | "running" | "completed" | "failed" | "stopped"
  | { other: string };

function statusFromString(s: string): SwarmLifecycleStatus {
  switch (s) {
    case "ready": case "running": case "completed":
    case "failed": case "stopped":
      return s;
    default:
      return { other: s }; // open-world: preserved, not dropped
  }
}

// The interesting part: rewriting on load, not a pure round-trip.
function recoverMemberStatus(
  persisted: SwarmLifecycleStatus,
  isHeadless: boolean,
): SwarmLifecycleStatus {
  if (persisted === "running") return "failed"; // "crashed" — nothing survives a restart
  if (persisted === "ready") return "stopped";   // don't resurrect stale ready clients
  if (isHeadless && !["completed", "failed", "stopped"].includes(persisted as string)) {
    return "failed";
  }
  return persisted;
}
```

The lesson to take from this pattern, independent of language: a
persistence boundary is a good place to ask "what part of this state
description was only ever true because a process was alive?" — and to
rewrite it explicitly on load rather than trusting it round-trips
unchanged.

---

## What this chapter left open

In the interest of not smoothing over gaps: the soft-interrupt injection
mechanics behind "queued and injected at safe points" were not
independently verified against `queue_soft_interrupt_for_session`'s
implementation — that claim is sourced from the design doc, not from
reading the injection code itself. Similarly, the swarm-specific
`event_history: Arc<RwLock<VecDeque<SwarmEvent>>>` mentioned earlier as a
contrast to the global bus's lack of replay is cited from Chapter 4's
territory, not independently re-opened here. And this chapter deliberately
didn't try to reproduce the literal tool-call parameter shape a caller uses
to trigger a DM vs. broadcast vs. status read — that's a schema-level
detail outside what was verified for this pass.
