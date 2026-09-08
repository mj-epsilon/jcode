# Phase 2 Analysis Notes — Chapter 7: Communication Topology

Verified personally by re-opening every cited file at the cited lines on
2026-08-12 against `master` @ `5ae238574`, including files the Phase 1 map
flagged as an unresolved gap. Where I add material not in the map, it's
flagged as new/additional. **The specific gap this pass was asked to close
— the `SwarmRole`/`SwarmLifecycleStatus` enum vs. the live
`SwarmMember.status: String`/`.role: String` translation — is resolved
below with real citations; conversion code does exist.**

---

## 0. Framing: two status vocabularies, and a document that's aware it's
half-superseded

Before the code, one framing fact the chapter should open with (already in
the map, re-confirmed by reading both docs' headers directly):
`docs/SWARM_ARCHITECTURE.md` (line 3-5) self-describes as "Largely
implemented... see SWARM_TASK_GRAPH.md for the DAG-first model that
supersedes the agent-first framing here; its staged comm migration is in
progress." `docs/SWARM_TASK_GRAPH.md` §8a (lines 407-475, read in full)
is the newer doc's own account of *why* the richer chat-style comm surface
(DMs + broadcast + channels + shared-context) is being cut down to a
leaner two-tier model. The chapter should present `SWARM_ARCHITECTURE.md`'s
Communication section as "what's still live today, described in the older
doc's vocabulary" and `SWARM_TASK_GRAPH.md` §8a as "the direction of
travel, with steps 1-2 already done, 3-4 pending" — not as two equally
-current descriptions.

`SWARM_TASK_GRAPH.md` §8a's staged migration list (verified, lines
464-474):
1. **Done.** Artifact dataflow: completion artifacts flow to dependents.
2. **Done.** Broadcast scoped to the sender's spawned subtree (incl.
   no-subscriber channel fallback and shared-context notifications);
   whole-swarm broadcast remains only as a coordinator escape hatch.
3. Migrate existing flows off channels/shared-context (pending).
4. Deprecate, then remove, the redundant chat primitives (pending).

Step 2 is directly verifiable in code (see §2 below) — I traced it myself
rather than trusting the doc's "Done" claim, and it checks out.

---

## 1. DMs, subtree broadcast, and channels — the actual routing code

**Where:** `crates/jcode-app-core/src/server/client_comm_message.rs`,
`handle_comm_message` — lines 103-300+ (I read through line 300; the
routing logic that matters for this chapter is fully contained in that
span). Not previously opened by the Phase 1 map at all — this is new
verification for this pass, and it's the concrete implementation behind
`SWARM_ARCHITECTURE.md`'s "DMs / subtree broadcast / channels" bullet list
(lines 199-205 of that doc) and `SWARM_TASK_GRAPH.md` §8a's "migration step
2" claim.

**How scope is decided (lines 217-223, verified verbatim):**
```rust
let scope = if resolved_to_session.is_some() {
    "dm"
} else if channel.is_some() {
    "channel"
} else {
    "broadcast"
};
```
A single `comm message` call is routed to one of three delivery shapes
based on which optional fields the caller filled in — `to_session` wins,
then `channel`, then it falls through to broadcast. This is the
"`message` already auto-routes among three of them" property
`SWARM_TASK_GRAPH.md` §8a complains about as evidence the surface has too
many overlapping primitives (line 423-424 of that doc) — a nice moment for
the chapter to let the design doc's own retrospective critique speak.

**DM target resolution — `resolve_dm_target_session`, lines 35-80**
(verified in full): a DM target can be an exact `session_id` (checked
first, lines 40-45) or a swarm-unique `friendly_name` (lines 47-60);
ambiguous friendly-name matches are a hard error naming every match (lines
67-78) rather than silently picking one.

**Subtree broadcast — lines 230-249, this is the concrete implementation
of migration step 2, verified verbatim:**
```rust
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
Two things worth calling out precisely: (1) this directly reuses
`swarm_is_self_or_ancestor` from `swarm.rs:76-85` — the exact same
authorization primitive the map already cited for "who may stop a session"
is *also* what scopes broadcast reach, a nice single-primitive-two-uses
callback; (2) `sender_is_coordinator` is checked against the bare string
`"coordinator"` (line 239, `member.role == "coordinator"`) — not the
`SwarmRole::Coordinator` enum variant. This is the first of several places
where the *live*, in-memory swarm logic is entirely string-based even
though a typed enum exists elsewhere (see §4 below — this is the load
-bearing evidence for that section, not a one-off).

**Channels — lines 251-272 (verified verbatim), including the deprecation
-aligned fallback:**
```rust
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
This is a direct use-site for `ChannelIndex` (the map's citation,
`jcode-swarm-core/src/lib.rs:235-340`, re-confirmed present and matches:
`by_swarm_channel`/`by_session` dual-map struct with a `members()` lookup
method). Worth noting: the code constructs a *fresh* `ChannelIndex` from
just the `by_swarm_channel` half on every call (line 255-258,
`by_session: HashMap::new()`) rather than holding a persistent
`ChannelIndex` instance — the persistent state is really just the raw
`channel_subscriptions: ChannelSubscriptions` map
(`Arc<RwLock<HashMap<String, HashMap<String, HashSet<String>>>>>`, type
alias at line 15), and `ChannelIndex` is used here as a one-shot lookup
helper wrapping it, not as the long-lived registry itself. Also worth
noting for the chapter as evidence channels are genuinely being
de-prioritized: an empty channel silently degrades to the subtree-broadcast
scope rather than erroring or doing nothing (lines 260-263) — channels
here behave as "a broadcast pre-filter that no-ops back to broadcast if
unused," which matches the "discouraged, prefer DMs" framing from
`SWARM_ARCHITECTURE.md:202-204`.

**Delivery mechanism, lines 274-300+ (read through 300):** every routed
target is delivered via `fanout_session_event(swarm_members, session_id,
ServerEvent::Notification { ... })` (line 290) — same fan-out helper cited
in the map for `broadcast_swarm_status_now`'s per-session send loop. Per
`SWARM_ARCHITECTURE.md:209-212`, these notifications are "queued as soft
interrupts and injected into running agents at safe points" — I did not
open the soft-interrupt injection code itself in this pass (it's imported
from `jcode_agent_runtime::SoftInterruptSource` at line 8 and referenced via
`queue_soft_interrupt_for_session` in the module's `use` list at line 4,
but the injection mechanics are out of scope for this chapter's citation
budget) — **flag as an open question**: the *claim* that notifications are
injected at "safe points" rather than interrupting mid-tool-call is from
the design doc, not independently verified against
`queue_soft_interrupt_for_session`'s implementation in this pass.

---

## 2. Status snapshot vs. summary vs. full context read — the three-tier
read model, precisely located

This closes the plan's explicit ask ("status snapshot vs. summary vs. full
-context reads") with real code, which the Phase 1 map did not open at
all (it only cited the design-doc prose at `SWARM_ARCHITECTURE.md:221-229`).
New in this pass: **`crates/jcode-app-core/src/server/comm_sync.rs`**.

The doc's three-way distinction (`SWARM_ARCHITECTURE.md:221-229`, verified
verbatim):
> "Status snapshot: lock-free member metadata plus current
> processing/tool snapshot. This must stay available even while the target
> agent is busy. Summary read: short activity feed (tool calls with
> intent, brief results, and optionally exposed thoughts). Full context
> read: explicit, heavy read of an agent's full context and tool outputs.
> This should be used sparingly to avoid context bloat."

This maps to three real handler functions, all in `comm_sync.rs`, all
re-read in full in this pass:

- **`handle_comm_status` — lines 249-324.** The "lock-free... even while
  the target agent is busy" claim is literally true of the implementation:
  the core snapshot (`AgentStatusSnapshot`, built at lines 304-320) is
  assembled entirely from the `swarm_members` registry (an
  `Arc<RwLock<HashMap<...>>>`, a data-structure lock, not the per-agent
  `Mutex<Agent>`) plus `file_touch` and `client_connections` state. It only
  *opportunistically* touches the agent mutex for provider/model name
  (lines 291-302), via `agent.try_lock()` — if that fails (agent busy), it
  just returns `(None, None)` for those two optional fields (lines
  296-298) rather than blocking or erroring. This is the one tier of the
  three that is designed to **never** block on a busy target.
- **`handle_comm_summary` — lines 194-243.** Requires locking the target
  agent (`agent.try_lock()`, line 218) to call
  `agent.get_tool_call_summaries(limit)` (line 219, default `limit = 10`,
  line 215). If the lock fails (agent busy), it does **not** silently
  degrade — it sends back an explicit error: `"Session '{}' is busy; try
  summary again shortly"` with `retry_after_secs: Some(1)` (lines 221-229).
  This is the middle tier: bounded/lightweight, but *can* fail transiently
  under load, unlike status.
- **`handle_comm_read_context` — lines 326-382.** Two layers of gating
  beyond the other two tiers: (1) `can_read_full_context` (lines 178-192,
  verified verbatim) — only the requester itself or a session whose
  `member.role == "coordinator"` may call this on another session's
  context (again: bare-string role comparison, not the `SwarmRole` enum —
  see §4); a non-coordinator, non-self caller gets an explicit error
  naming the restriction (lines 346-353: "Only the coordinator, worktree
  manager, or the target session may read full context. Use summary for
  lightweight access."). (2) Same `try_lock`-or-busy-error pattern as
  summary (lines 357-369), but returns `agent.get_history()` — the full
  message history — rather than a capped tool-call digest.

**Shared gate for all three:** `ensure_same_swarm_access` (lines 144-176,
verified verbatim) — every one of the three handlers requires requester and
target to share a `swarm_id` before anything else runs; this is the base
authorization layer under all inter-agent reads, not specific to any one
tier.

This is a genuinely clean three-tier design to feature as a worked example:
same underlying data (an `Agent` behind a `Mutex`, plus a lock-free
metadata registry) exposed at three different cost/availability points —
always-available-but-shallow, cheap-but-can-fail-busy,
expensive-and-permission-gated. Good candidate for the chapter's own
"communication topology" diagram to include as a third axis (who can read
what, at what cost) alongside "who can send what to whom" (DM/broadcast/
channel from §1).

---

## 3. The global event bus (`jcode-base/src/bus.rs`)

Re-read in full (lines 390-607 of the relevant span) to independently
confirm the map's characterization. All of the following matches the map
exactly — no corrections needed here, just direct re-verification:

- **`BusEvent` enum — lines 394-466** (~30 variants), re-confirmed the
  swarm-adjacent ones: `BatchProgress` (399), `FileTouch` (401),
  `SwarmOutputTail` (403), `SwarmAwaitCompleted` (409) — this is the exact
  type published from `comm_await.rs:196`, confirming the cross-chapter
  link (Chapter 3 pattern 3's `finalize_await` hands off to this bus).
- **`Bus` struct — lines 468-474**: just `sender: broadcast::Sender<BusEvent>`
  plus private debounce state. Confirms "the entire bus is one tokio
  broadcast channel sender."
- **`Bus::global() — lines 499-502`**, re-verified verbatim:
  ```rust
  pub fn global() -> &'static Bus {
      static INSTANCE: OnceLock<Bus> = OnceLock::new();
      INSTANCE.get_or_init(Bus::new)
  }
  ```
  Process-wide lazy singleton.
- **`Bus::new() — lines 511-519`**: `broadcast::channel(256)` — fixed
  256-slot ring buffer. This is why `comm_await.rs`'s handling of
  `RecvError::Lagged` (Chapter 3, Pattern 3) matters in practice: any
  subscriber that falls more than 256 unconsumed events behind will start
  missing events, and each subscriber lags independently (a slow
  subscriber doesn't block the publisher or other subscribers).
- **`Bus::publish` — lines 525-533**: `let _ = self.sender.send(event);` —
  deliberately ignores the "zero receivers" error case (nobody listening
  isn't a failure for a fire-and-publish bus). Special-cases
  `UpdateStatus` by additionally caching it in a separate `Mutex` (lines
  526-531, backed by `latest_update_status()` at lines 478-481) — a
  documented workaround for "broadcast channels have no replay/history":
  a late subscriber can call `Bus::latest_update_status()` (lines 535-540)
  to poll the last-known value instead of having to have been subscribed
  at publish time.
- **`Bus::publish_models_updated` — lines 542-606**: a second, independent
  750ms-debounce implementation (`MODELS_UPDATED_DEBOUNCE`, line 476) —
  confirms the map's "this coalescing pattern recurs" note; structurally
  the same idea as `swarm.rs`'s status-broadcast debounce
  (`broadcast_swarm_status`, cited below) but implemented completely
  separately, with its own per-instance debounce state (a documented
  choice — the comment at lines 470-472 explains it's per-instance rather
  than a global static specifically so tests can exercise coalescing on an
  isolated bus without racing other tests on the global one).

**No replay/history vs. `swarm.rs`'s `event_history`:** worth an explicit
contrast in the chapter (map already flagged this, re-confirmed): the
global `Bus` has no general replay mechanism (just the one hand-rolled
`UpdateStatus` special case above), whereas the swarm-specific event stream
keeps `event_history: Arc<RwLock<VecDeque<SwarmEvent>>>` (cited in
`runtime.rs:107` per the map, not independently re-opened in this pass —
flag as relying on the map's citation for that one field, everything else
in this section was independently re-verified).

---

## 4. THE GAP: `SwarmRole`/`SwarmLifecycleStatus` vs. `SwarmMember.status`/
`.role` — **conversion code exists, and it's not the identity function**

The Phase 1 map explicitly flagged this as unresolved ("the actual
translation/conversion code between them (if any) was not located").
**I searched for it and found it. It exists, it's more interesting than a
simple round-trip, and it deserves a full callout in the chapter, not a
footnote.**

### The two representations, re-confirmed

- **Live, in-memory:** `SwarmMember` — `crates/jcode-app-core/src/server/
  state.rs:186-239` (re-read in full). `status: String` (line 204, doc
  comment: "Lifecycle status (ready, running, completed, failed, stopped,
  etc.)"), `role: String` (line 218, doc comment: "Role: \"agent\" or
  \"coordinator\""). Every piece of *live* logic I found in this pass
  compares these as bare strings against literal string constants — see
  `member.role == "coordinator"` in `client_comm_message.rs:239` and
  `comm_sync.rs:190`, `member.status == "running"` in `comm_sync.rs:288`,
  and (from the map, not re-opened this pass) similar comparisons
  throughout `swarm.rs`.
- **Persisted, typed:** `SwarmMemberRecord` — `crates/jcode-swarm-core/
  src/lib.rs:215-231` (re-read). `status: SwarmLifecycleStatus` (13 named
  variants + `Other(String)` catch-all, lines 136-151), `role: SwarmRole`
  (`Agent`/`Coordinator`/`Other(String)`, lines 91-95). Doc comment on the
  struct (line 213): "Durable, persistable portion of a swarm member."

### The conversion code

**`SwarmMember::durable_record(&self) -> SwarmMemberRecord` —
`crates/jcode-app-core/src/server/state.rs:243-258`, verified verbatim:**
```rust
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
`SwarmLifecycleStatus::from(String)` and `SwarmRole::from(String)` are
hand-written `impl From<String> for ...` blocks in `jcode-swarm-core/src/
lib.rs` — `SwarmRole`'s at **lines 107-115**, `SwarmLifecycleStatus`'s at
**lines 174-193** (both re-read verbatim). Both match on the string against
every known literal (e.g. `"ready" => Self::Ready`, `"running" =>
Self::Running`, ... `"todo" => Self::Todo`) and fall through to
`Self::Other(value)` for anything unrecognized (line 112 for `SwarmRole`,
line 190 for `SwarmLifecycleStatus`) — so the conversion is total (never
panics/errors), open-world (an unrecognized string is preserved, not
dropped), but does mean typos or new ad-hoc status strings silently become
`Other("...")` rather than being caught as bugs at the type level.

**`SwarmMember::from_record(record: SwarmMemberRecord, event_tx) -> Self`
— `state.rs:270-296`, the reverse direction, verified verbatim:**
```rust
status: record.status.as_str().into_owned(),
...
role: record.role.as_str().into_owned(),
```
`as_str(&self) -> Cow<'_, str>` is defined for both enums (`SwarmRole` at
`jcode-swarm-core/src/lib.rs:98-104`, `SwarmLifecycleStatus` at
`154-171`) — a plain match returning the canonical lowercase string for
each named variant (e.g. `Self::RunningStale => Cow::Borrowed("running_stale")`)
or the wrapped string for `Other`. So the round trip
`String -> enum -> String` is lossless for any string that maps to a named
variant (it always maps back to the *same* canonical string, even if the
input had different casing/whitespace than the canonical form — worth
flagging: `From<String>` matches exact lowercase literals only, so e.g.
`"Ready"` would fall into `Other("Ready")`, not `Ready` — this is a real,
if narrow, footgun the type-level round trip doesn't fully protect
against), and identity for `Other(String)` values.

### Where these two functions are actually called — confirms this is live,
not dead, code

`crates/jcode-app-core/src/server/swarm_persistence.rs`, verified by
re-reading lines 300-620:
- **`to_persisted_member` (lines 323-332)** calls `member.durable_record()`
  (line 329) — this is the save path, called from `persist_swarm_state`
  (line 583, confirmed present via grep, `.map(|member|
  to_persisted_member(member, snapshot_unix_ms))` at line 616).
- **`from_persisted_member` (lines 398-441)** calls `SwarmMember::from_record`
  (line 426) — this is the load path.

### A genuinely interesting nuance beyond "conversion exists": the round
trip is **not** a pure identity function in practice, because of
`recover_member_status`

**`recover_member_status` — `swarm_persistence.rs:341-390`, verified
verbatim.** This function runs *between* loading a persisted
`SwarmMemberRecord` and reconstructing the live `SwarmMember`
(`from_persisted_member` calls it at line 408, before calling
`SwarmMember::from_record` at line 426) and deliberately rewrites certain
statuses on load, because a live process context (the running task, the
event channel) cannot survive a server restart even though the *status
string* it left behind can:
- A persisted `Running` status becomes `Crashed` on load (lines 346-351) —
  nothing can genuinely still be "running" after the server that was
  running it restarted.
- A persisted `Ready` status becomes `Stopped` on load (lines 358-370),
  with a doc comment explaining a real historical bug this fixes: "No
  client or headless process survives a server restart... This previously
  resurrected hundreds of detached historical clients as ready on every
  reload."
- A headless member in any non-terminal-ish status becomes `Crashed`
  (lines 374-387, guarded to skip `Completed`/`Done`/`Failed`/`Stopped`,
  which are legitimately final and safe to keep as-is).

This means the conversion the map asked about isn't just "does a
translation layer exist" — it's "the translation layer is where the
codebase enforces that persisted status can never lie about surviving a
restart." That's a strong, concrete Chapter 7 (and arguably Chapter 5)
exhibit: the typed enum's only real behavioral job in this codebase, beyond
serialization, is to be the thing `recover_member_status` pattern-matches
on to apply this restart-recovery policy — something a bare `String` could
technically also support via string comparison, but the enum makes the
match exhaustive-checked by the compiler (`SwarmLifecycleStatus::Running`
etc. are compile-time-known variants, not string literals that could typo).

### Verdict for the chapter

This is **not** a "no conversion exists" finding (the map left that door
open as a possibility) — conversion code exists, is exercised on every
swarm-state save/load cycle, and does meaningful recovery work beyond a
naive round-trip. The chapter should present this as a deliberate
two-tier design: **live code operates on loose, ergonomic `String` fields
for speed of iteration and because in-process comparisons against string
literals are cheap and don't need serialization**, while **the
persistence boundary opts into a typed, exhaustively-matchable enum
specifically because that's where a bug (silently losing "was this
actually still running") would be expensive and hard to notice** — a nice
concrete instance of "use the type system where correctness matters most,
not uniformly everywhere," worth stating as an explicit design lesson.

---

## 5. Cross-reference: plan/status fan-out is targeted unicast-per-member,
contrasted with the bus's single broadcast channel

Re-confirmed from the map by re-reading `swarm.rs:685-970` directly (new
in this pass — the map cited this material but I re-verified the exact
code rather than trusting the paraphrase):

- **`broadcast_swarm_status_now` (lines 685-740)** and
  **`broadcast_swarm_status` (lines 742-812)**: the debounce wrapper
  (`742-812`) checks member count against
  `swarm_status_debounce_member_threshold()`; below threshold, sends
  immediately (`758-761`); at/above threshold, coalesces via a
  `tokio::spawn`'d loop keyed by `pending_swarm_status_broadcasts()` (an
  `OnceLock<StdMutex<HashMap<...>>>` process-global) so N rapid changes in
  a busy swarm collapse into one flush every
  `swarm_status_debounce_ms()`. The actual send (`broadcast_swarm_status_now`,
  line 737-739) loops over each affected session id and calls
  `fanout_session_event(swarm_members, &sid, event.clone())` — i.e. it is
  logically "broadcast" in effect (many recipients) but physically it's a
  loop of *individual per-member sends* through each member's own
  `event_tx`/`event_txs`, not one send on a shared channel.
- **`broadcast_swarm_plan_with_previous` (lines 836-920)**: same shape —
  delivery loop at lines 902-908 does `member.event_tx.send(event.clone())`
  per participant. Doc comment (lines 814-817, re-verified verbatim):
  "Plan snapshots are sent to explicit plan participants. If a plan has no
  participants yet, fall back to all current swarm members."
- **`send_swarm_plan_to_session` (lines 926-970)**: the single-recipient
  variant for reconnect/resume — doc comment (922-925) explains it exists
  because "reconnecting clients need an immediate snapshot rather than
  waiting for the next mutation."

This is worth stating explicitly as a *topology* contrast for the chapter:
`jcode-base::bus::Bus` is one `tokio::sync::broadcast` channel that
everyone subscribes to (true broadcast, one send reaches every current
subscriber). Swarm plan/status fan-out is the opposite shape: each member
holds its *own* `mpsc` sender(s) (`event_tx`/`event_txs` on `SwarmMember`,
`state.rs:194,196`), and "broadcasting" to a swarm means iterating the
member list and sending to each one individually — a fan-out loop over
point-to-point channels, not a shared broadcast primitive. Good moment for
a TS analogy: the global `Bus` ~ one `EventEmitter`/pub-sub topic every
listener subscribes to; the swarm status/plan fan-out ~ a `Map<sessionId,
sendFn>` you iterate and call individually — structurally a "fan-out
loop," not a "broadcast primitive," even though the end effect (many
recipients get the same message) looks the same from a distance.

---

## Open questions / things I could NOT verify (explicit, not papered over)

1. **Soft-interrupt injection mechanics** (`SoftInterruptSource`,
   `queue_soft_interrupt_for_session`) — I confirmed these are imported
   and called from `client_comm_message.rs` but did not open their
   implementation. The design doc's claim that notifications are "queued
   ... and injected ... at safe points" (`SWARM_ARCHITECTURE.md:211-212`)
   is therefore design-doc-sourced, not independently code-verified in
   this pass.
2. **`event_history: Arc<RwLock<VecDeque<SwarmEvent>>>`** — cited only via
   the map's earlier reference to `runtime.rs:107`; I did not re-open
   `runtime.rs` in this pass (Chapter 4's territory, already covered
   there per the map). If Chapter 7 wants to feature the "no replay in
   global Bus vs. replay in swarm event_history" contrast with its own
   citation (not borrowed from the map), that field should be re-opened
   directly.
3. **Exact current `swarm` tool action/parameter names** — same gap the
   map already flagged (its "Known gap #1"): I did not open the live
   `swarm` tool's parameter schema in this pass either. Not directly
   needed for this chapter's DM/broadcast/channel/bus/read-tier material
   (all of which I verified via the handler functions directly, not the
   tool schema), but if Chapter 7 wants to quote the literal tool-call
   shape a caller uses to trigger `comm_message`/`comm_status`/etc., that
   still needs its own verification pass.
4. **Reparenting-on-departure** (`SWARM_ARCHITECTURE.md:40-45`'s claim
   that a departing member's children reparent to their grandparent) —
   this is Chapter 2 territory per the map, not re-verified here; flagging
   only because it's adjacent to "communication topology" (who a
   broadcast/DM reaches depends on the live spawn tree) and Chapter 7
   should not casually assert reparenting mechanics without Chapter 2's
   own verification of `remove_session_from_swarm`
   (`swarm.rs:995-1219`, still unread line-by-line as of this pass).
