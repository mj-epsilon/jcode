# Phase 2 Notes — Chapter 2: Anatomy of a spawn

Status: every citation below was opened and read directly by this Phase 2
pass on 2026-08-12 against the current source tree (master, no new commits
since the Phase 1 map's `5ae238574`). This chapter closes three of the
map's flagged gaps (live tool schema, `run_swarm_task` reachability
correction, and the `report_back_to_session_id` set-site) with fresh
citations, not the map's guesses.

---

## 0. Critical scoping correction before anything else

**There are two different "spawn a child agent" code paths in this
codebase, and only one of them is what a live LLM agent actually triggers.**
This must be stated up front in the chapter, or the reader will come away
believing `run_swarm_task`/`run_swarm_message` (the `try_join_all`
planner→fan-out→fan-in flow in `swarm.rs`) is what happens when an agent
calls the `swarm` tool. It is not.

- **The real, agent-facing spawn path** (this chapter's actual subject):
  `swarm` tool, `action: "spawn"` → `Request::CommSpawn` → server-side
  `handle_comm_spawn` → `spawn_swarm_agent`
  (`crates/jcode-app-core/src/server/comm_session.rs:557-827`).
- **A separate, debug-only path**: `run_swarm_task`/`run_swarm_message`
  (`crates/jcode-app-core/src/server/swarm.rs:1528-1701`) creates a
  `Session`+`Agent` pair too (`Session::create` at line 1548,
  `Agent::new_with_session` at line 1583), but I traced every caller of
  `run_swarm_message` and found exactly two, both debug-socket string
  commands, not the `swarm` tool:
  ```rust
  // crates/jcode-app-core/src/server/debug_command_exec.rs:132-139
  if trimmed.starts_with("swarm_message:") {
      let msg = trimmed.strip_prefix("swarm_message:").unwrap_or("").trim();
      ...
      let final_text = super::run_swarm_message(agent.clone(), msg).await?;
      return Ok(final_text);
  }
  ```
  ```rust
  // crates/jcode-app-core/src/server/debug_jobs.rs:77-95
  if trimmed.starts_with("swarm_message_async:") {
      ...
      let result = super::run_swarm_message(agent.clone(), &msg).await;
  ```
  I also confirmed the `swarm` tool's own `"message"` action
  (`crates/jcode-app-core/src/tool/communicate.rs:2261-2276`) does
  something else entirely (routes a DM/broadcast/channel post — its own
  comment says so verbatim: *"`message` is the general-purpose send: it
  routes by the fields provided... With `to_session` it acts as a DM, with
  `channel` it posts to that channel, and with neither it broadcasts to the
  sender's spawned subtree"*) and never calls `run_swarm_message`.

**Recommendation for the chapter**: use `spawn_swarm_agent` /
`handle_comm_spawn` as the one true "anatomy of a spawn" walkthrough. If the
chapter wants to mention `run_swarm_task`'s `Session::create` +
`Agent::new_with_session` pairing as a second, simpler illustration of "what
creating a session even looks like," it's fine to show the snippet, but
must be labeled as **a debug/dev-tooling code path**, not the mechanism a
production agent uses to fan out work. (This correction also matters for
whoever writes Chapter 3 — the plan's "flagship `try_join_all`" framing
needs the same caveat.)

## 1. The live `swarm` tool schema — CLOSES MAP GAP #3 (verified, not "proposed")

File: `crates/jcode-app-core/src/tool/communicate.rs` (3351 lines).
`struct CommunicateTool` implements `Tool`:
```rust
// crates/jcode-app-core/src/tool/communicate.rs:1939-1946
impl Tool for CommunicateTool {
    fn name(&self) -> &str {
        "swarm"
    }
    fn description(&self) -> &str { &self.description }
```
Full current action enum, read directly from the schema (not from
`SWARM_TASK_GRAPH.md`'s "proposed tool surface"):
```rust
// crates/jcode-app-core/src/tool/communicate.rs:1954-1963
"action": {
    "type": "string",
    "enum": ["share", "share_append", "read", "message", "broadcast", "dm", "channel", "list", "list_channels", "channel_members",
             "propose_plan", "approve_plan", "reject_plan", "spawn", "stop", "assign_role",
             "status", "report", "plan_status", "summary", "read_context", "resync_plan", "assign_task", "assign_next", "fill_slots", "run_plan", "cleanup",
             "task_graph", "expand_node", "complete_node", "inject_gap",
             "start", "start_task", "wake", "resume", "retry", "reassign", "replace", "salvage",
             "subscribe_channel", "unsubscribe_channel", "await_members", "list_models"],
    "description": "Action. spawn requires label and should include prompt. list_models shows available models/routes."
},
```
This confirms `SWARM_TASK_GRAPH.md`'s §8 "proposed" names
(`expand_node`/`complete_node`/`inject_gap`/`task_graph`) are in fact live,
current action names — the doc's hedge ("proposed") is now stale; the
handbook can assert these as real, current API surface with this citation.

Spawn-relevant parameters (all read directly from the same schema):
```rust
// crates/jcode-app-core/src/tool/communicate.rs:2009-2021, 2043-2047
"label": {
    "type": "string", "minLength": 1,
    "description": "Required for spawn. Short label shown on the agent's chip (e.g. 'api reviewer')."
},
"working_dir": { "type": "string", "description": "Optional working directory for spawn." },
"prompt": {
    "type": "string",
    "description": "Initial task/instructions for spawn. Spawning without it creates an idle agent."
},
"initial_message": { "type": "string", "description": "Alias of prompt for spawn; wins when both are set." },
...
"spawn_mode": {
    "type": "string",
    "enum": ["visible", "headless", "inline", "auto"],
    "description": "Spawn UI mode: visible terminal, headless, inline gallery, or auto. Defaults to inline."
},
```
`spawn_mode`'s enum values map 1:1 to the `SwarmSpawnMode` Rust enum (see
§3 below) — good "the tool schema and the Rust type agree" exhibit.

The schema enforces `label` as required **only** for `action: "spawn"**,
via a JSON-Schema `anyOf` (not the flat top-level `required` array) with an
inline comment explaining a real cross-provider compatibility constraint:
```rust
// crates/jcode-app-core/src/tool/communicate.rs:2154-2187
// `swarm` is a multi-action tool, so putting `label` in the top-level
// `required` array would incorrectly require it for read/list/message and
// every other action. Use mutually exclusive action branches instead...
// `anyOf` object branches are supported by our provider schema adapters
// and avoid the less-portable JSON Schema `if`/`then` keywords.
...
schema["anyOf"] = json!([
    {
        "type": "object",
        "required": ["action", "label"],
        "properties": {
            "action": { "type": "string", "enum": ["spawn"] },
            // Gemini validates that every `required` name is defined in
            // the same object's `properties` and rejects the whole
            // request otherwise (issue #655), so declare `label` here
            // instead of relying on the parent schema's declaration.
            "label": { "type": "string", "minLength": 1 }
        }
    },
    { "type": "object", "required": ["action"], "properties": { "action": { "type": "string", "enum": non_spawn_actions } } }
]);
```
Nice teaching aside: a real, dated, cross-provider-compatibility bug
(`issue #655`, a Gemini schema-validation quirk) shaped this exact bit of
JSON Schema — worth a short callout that tool schemas for multi-model
LLM harnesses have their own portability constraints, similar in spirit to
writing DOM code that has to work across browsers.

Dispatch — the `"spawn"` action handler, which is the tool-layer half of
the spawn path:
```rust
// crates/jcode-app-core/src/tool/communicate.rs:2720-2751
"spawn" => {
    let label = params.required_spawn_label()?;
    let request = Request::CommSpawn {
        id: REQUEST_ID,
        session_id: ctx.session_id.clone(),
        working_dir: params.working_dir.clone(),
        initial_message: params.spawn_initial_message(),
        request_nonce: None,
        spawn_mode: params.spawn_mode.clone(),
        model: params.model.clone(),
        effort: params.effort.clone(),
        label: Some(label),
    };
    match send_request(request).await {
        Ok(ServerEvent::CommSpawnResponse { new_session_id, .. }) if !new_session_id.is_empty() => {
            Ok(ToolOutput::new(format!("Spawned new agent: {}", new_session_id)))
        }
        ...
    }
}
```
**[RUST plain-English]**: the tool-execution code doesn't touch swarm state
directly — it builds a `Request` enum value and calls `send_request(...)
.await`, an async round-trip over an internal request/response channel to
a central server-side handler. This is a deliberate layering: the code that
parses/validates the LLM's tool-call JSON is decoupled from the code that
actually mutates the shared swarm registries, communicating only through a
typed message, not shared references. **TS-equivalent mapping**: this is
structurally like a client posting a typed action object to a single
dispatcher/reducer (Redux-style `dispatch(action)`, or a `postMessage` to a
worker that owns the real state) rather than the tool handler reaching into
a shared `Map` itself.

## 2. `spawn_swarm_agent` — the real spawn path (comm_session.rs:557-827)

Re-read in full (271 lines). Signature:
```rust
// crates/jcode-app-core/src/server/comm_session.rs:557-579
pub(super) async fn spawn_swarm_agent(
    req_session_id: &str,
    swarm_id: &str,
    working_dir: Option<String>,
    initial_message: Option<String>,
    spawn_mode: Option<SwarmSpawnMode>,
    requested_model: Option<String>,
    requested_effort: Option<String>,
    label: Option<String>,
    sessions: &SessionAgents,
    ... // 10 more shared-state/channel params
) -> anyhow::Result<String> {
```
Body, verified step by step against the actual source (not paraphrased from
the map):

1. **Resolve working dir + auth identity to inherit** — lines 580-588:
   `resolve_spawn_working_dir` and `resolve_coordinator_spawn_identity`
   (the latter is what lets a spawned child inherit the parent's exact
   provider/model/auth route rather than falling back to config defaults).
2. **Resolve the spawn UI mode** — line 591:
   `let resolved_spawn_mode = spawn_mode.unwrap_or(agents_config.swarm_spawn_mode);`
   — falls back to the configured default (`Inline`, see §3) when the tool
   call didn't specify one.
3. **Prepare the startup message** — lines 618-620: if an `initial_message`
   was given, it gets wrapped with
   `append_swarm_completion_report_instructions` (from `jcode-swarm-core`,
   not re-verified line-by-line this pass, but confirmed to exist by the
   Phase 1 map at `jcode-swarm-core/src/lib.rs:355-375`) — this appends a
   system-reminder telling the new worker to call `swarm report` before
   finishing.
4. **Attempt a visible spawn, else fall back to headless** — lines 622-695:
   ```rust
   // crates/jcode-app-core/src/server/comm_session.rs:622-652
   let visible_spawn = match resolved_spawn_mode {
       // Inline workers run in-process like headless ones; the difference is
       // purely how the coordinator renders them (a live inline gallery).
       SwarmSpawnMode::Headless | SwarmSpawnMode::Inline => {
           Err(anyhow::anyhow!("headless spawn requested"))
       }
       SwarmSpawnMode::Visible | SwarmSpawnMode::Auto => prepare_visible_spawn_session(...),
   };
   ```
   i.e. `Headless`/`Inline` (the default) skip the visible-window attempt
   entirely by construction — `Err(...)` is used here purely as a
   control-flow shortcut into the same fallback branch visible-spawn
   failures use, not a real error. The fallback:
   ```rust
   // crates/jcode-app-core/src/server/comm_session.rs:654-695
   let (new_session_id, is_headless_fallback) = match visible_spawn {
       Ok((new_session_id, true)) => Ok((new_session_id, false)),
       Ok((_, false)) | Err(_) => {
           ...
           create_headless_session(..., Some(req_session_id.to_string()), ...).await...
       }
   }?;
   ```
5. **Register the new member as a plan participant, conditionally** —
   lines 698-706: only touches `plan.participants` if the swarm's shared
   plan already has items or participants — an empty plan is left alone.
6. **Broadcast the plan update** — lines 708-715 (`broadcast_swarm_plan`).
7. **Register the `SwarmMember` record** — lines 716-730, but **only for
   the non-headless-fallback branch**:
   ```rust
   // crates/jcode-app-core/src/server/comm_session.rs:716-730
   if !is_headless_fallback {
       register_visible_spawned_member(
           &new_session_id, swarm_id, resolved_working_dir.as_deref(),
           startup_message.is_some(), Some(req_session_id),
           swarm_members, swarms_by_id, event_history, event_counter, swarm_event_tx,
       ).await;
   }
   ```
   Open question I could not fully close this pass: for the headless path
   (the common/default case), member registration must happen *inside*
   `create_headless_session` itself (`crates/jcode-app-core/src/server/headless.rs`)
   rather than here — I confirmed `create_headless_session`'s signature
   accepts a `report_back_to_session_id: Option<String>` parameter (see §4
   below) and is called with `Some(req_session_id.to_string())`, which
   proves the parent edge is set correctly for headless spawns too, but I
   did not read the full body of `headless.rs` to find its own
   `SwarmMember` construction site. Safe to cite the parameter-passing
   proof above; do not claim a specific line number for headless member
   construction without opening `headless.rs` further.
8. **Task label** — line 736: `set_member_task_label(&new_session_id,
   label_text, swarm_members)`, where `label_text` prefers the explicit
   `label` param, falling back to the raw `initial_message`.
9. **Persist state** — line 744, before anything below fires.
10. **Conditional fire-and-forget task** — lines 746-824, only when
    `is_headless_fallback && startup_message.is_some()`:
    ```rust
    // crates/jcode-app-core/src/server/comm_session.rs:772-822
    tokio::spawn(async move {
        update_member_status(&sid_clone, "running", ..., ...).await;
        let event_tx = super::session_event_fanout_sender(...);
        let start_message_index = { let agent = agent_arc.lock().await; agent.message_count() };
        let result = process_message_streaming_mpsc(Arc::clone(&agent_arc), &initial_msg, vec![], None, event_tx).await;
        let completion_report = if result.is_ok() { ... agent.latest_assistant_text_after(start_message_index) } else { None };
        let (new_status, new_detail) = match result {
            Ok(()) => ("ready", None),
            Err(ref error) => ("failed", Some(truncate_detail(&error.to_string(), 120))),
        };
        update_member_status_with_report(&sid_clone, new_status, new_detail, completion_report, ...).await;
    });
    ```
    Confirms the map's corrected citation exactly: the `tokio::spawn(` call
    is line 772, closing brace at line 822. This block does NOT fire for
    visible spawns, nor for headless spawns with no initial message — those
    return synchronously with just the new session id
    (`Ok(new_session_id)`, line 826) and whatever runs the first turn
    happens elsewhere (client-driven).
    **[RUST plain-English]**: `tokio::spawn(async move { ... })` starts a
    concurrently-running task and immediately hands back a `JoinHandle`;
    here that handle is dropped (never stored, never `.await`ed), which is
    exactly what "fire-and-forget" means in async Rust — the outer function
    returns `Ok(new_session_id)` without waiting for the inner task, and
    the *only* way a caller ever learns how that turn went is by later
    reading the member's status/`latest_completion_report`, never via a
    `Result` returned to whoever spawned it. **TS-equivalent mapping**:
    calling an `async function` without `await`ing it — a "floating
    promise," which most TS linters flag by default specifically because
    errors silently vanish otherwise. This is a genuine, real tradeoff of
    the jcode design worth naming plainly, not softening.

## 3. `SwarmSpawnMode` — the UI/rendering mode-gate (mode axis #1 of 2)

```rust
// crates/jcode-config-types/src/lib.rs:643-657
/// How swarm-created agents should be spawned.
#[derive(Debug, Clone, Copy, Serialize, Deserialize, PartialEq, Eq, Default)]
#[serde(rename_all = "lowercase")]
pub enum SwarmSpawnMode {
    /// Open a visible/headed terminal window. This was the historical default.
    Visible,
    /// Create the worker in-process without opening a terminal window.
    Headless,
    /// Like headless (no terminal window), but the coordinator renders a live
    /// inline gallery viewport of each worker's streaming output.
    #[default]
    Inline,
    /// Try visible first and fall back to headless if a window cannot be opened.
    Auto,
}
```
This is **not** the "ad hoc / light / deep" mode axis named in the chapter
brief — it's purely about *how the worker's UI is rendered/attached*
(separate terminal window vs. in-process). The important plain-English
point for the reader: `Headless` and `Inline` both mean "the child agent
runs as an async task inside the *same OS process* as the parent server" —
there is no new process or OS thread for the common case. Only `Visible`
(and `Auto`'s fallback-from-visible) launches a genuinely separate
terminal/process. **TS-equivalent mapping**: `Headless`/`Inline` ~ spawning
another concurrently-running async job in the same Node process (tracked in
a map, e.g. `Map<sessionId, Promise>` — not a `worker_thread`); `Visible`
~ actually spawning a subprocess (`child_process.spawn`, a new OS-level
window/process), which is a meaningfully different primitive and worth
contrasting explicitly.

## 4. The "ad hoc / light / deep" mode axis (mode axis #2 of 2) and the member cap — THE core of this chapter

This is the actual mode-gate named in the plan brief, and it's a
**different Rust concept than `SwarmSpawnMode`** — worth a very explicit
"these are two separate axes, don't conflate them" callout, since both are
casually called "mode" in prose.

### What "ad hoc / light / deep" actually is, mechanically

There is no `SwarmMode` Rust enum with three variants named exactly that.
Instead:

- **"deep"** is a **reasoning-effort sentinel string**, `"swarm-deep"`,
  checked via a helper function:
  ```rust
  // crates/jcode-base/src/prompt.rs:129-131
  pub fn is_deep_swarm_effort(effort: &str) -> bool {
      effort.trim().eq_ignore_ascii_case(SWARM_DEEP_EFFORT)
  }
  ```
  (`SWARM_DEEP_EFFORT = "swarm-deep"`, `prompt.rs:110`). There's a sibling,
  `"swarm"` (`SWARM_EFFORT`), and `is_swarm_effort` which matches either —
  this is the **"light"** rung: effort `"swarm"` enables swarm orchestration
  generally without the recursive/DAG-gated machinery. **"ad hoc"** is
  simply *neither* sentinel — ordinary spawning with no special effort
  rung engaged at all, still allowed because spawning itself isn't gated by
  effort, only *recursive* (grandchild) spawning is (see below). This
  mapping (`ad hoc` = default, `light` = effort `"swarm"`, `deep` = effort
  `"swarm-deep"`) is my own synthesis from reading `prompt.rs:110-131` plus
  the gating code below — it is not spelled out as a single named enum
  anywhere in the code, so the handbook should present it as "three
  operating modes distinguished by the session's recorded reasoning
  effort," not as a literal `enum Mode { AdHoc, Light, Deep }` that doesn't
  exist.
- Each session's current effort is tracked in a small, deliberately
  lock-cheap side-table specifically to avoid a deadlock:
  ```rust
  // crates/jcode-app-core/src/session_effort.rs:1-19
  //! The swarm task-graph seed handler runs on the server socket thread while the
  //! *seeding* agent is blocked inside its `swarm` tool call holding its own agent
  //! lock. That means the handler cannot read the seeder's effort via the agent
  //! mutex without deadlocking. This tiny side-table is updated whenever an agent's
  //! effort changes (cheap string writes) so server handlers can learn a session's
  //! effort by id without taking any agent lock.
  static SESSION_EFFORTS: LazyLock<RwLock<HashMap<String, String>>> = ...;
  pub fn session_effort(session_id: &str) -> Option<String> { ... }
  ```
  Good, concrete "why a side-index instead of the obvious lookup" exhibit —
  directly parallel to the `BACKGROUND_TOOL_SIGNALS` side-registry pattern
  the Phase 1 map documented for Chapter 5 (state.rs:15-30); worth a
  cross-reference if Chapter 5 exists in the final handbook.

### The actual gate: `ensure_spawn_coordinator_swarm`, comm_session.rs:1234-1431

I read this function in full (198 lines) — it is the single best
"anatomy of the mode-gate + member cap" citation in the codebase, better
and more precise than the map's summary. Three checks, in order, each one
independently gating a spawn attempt:

**(a) The mode gate itself** — only non-root sessions are checked (roots
can always spawn a first generation):
```rust
// crates/jcode-app-core/src/server/comm_session.rs:1330-1348
// Light and ad hoc swarms are deliberately one-level fan-out: only the root
// session may create workers. Recursive spawning is an explicit deep-swarm
// capability, keyed from the root's effort rather than the requesting
// child's effort so a worker cannot opt itself into unbounded growth.
if !is_root {
    let root_is_deep = crate::session_effort::session_effort(&root_session_id)
        .as_deref()
        .is_some_and(crate::prompt::is_deep_swarm_effort);
    if !root_is_deep {
        let _ = client_event_tx.send(ServerEvent::Error {
            id,
            message: format!(
                "Recursive swarm spawning is disabled for light and ad hoc swarms. Only the root session ({root_session_id}) may spawn agents unless that root is running in swarm-deep mode."
            ),
            retry_after_secs: None,
        });
        return None;
    }
}
```
**Design point worth calling out explicitly**: the check reads the **root's**
effort (`root_session_id`, found by walking `swarm_ancestors(...).last()`,
line 1266-1269), not the requesting child's own effort — the doc comment
states the reason plainly: *"so a worker cannot opt itself into unbounded
growth."* A malicious or confused child agent cannot self-elevate into
recursive spawning by claiming a high effort locally; only the root's
recorded state matters. This is a genuinely nice access-control design
detail — the equivalent TS teaching point is "authorize against a
capability derived from the root of the call chain, not a value the callee
controls."

**(b) The absolute hard cap**:
```rust
// crates/jcode-app-core/src/server/comm_session.rs:1350-1361
// Keep an absolute hard ceiling even when the configurable limit is disabled.
if live_member_count >= super::MAX_SWARM_MEMBERS {
    let _ = client_event_tx.send(ServerEvent::Error {
        id,
        message: format!(
            "Swarm member limit reached (hard max {}). This swarm already has {live_member_count} live members; it cannot spawn more. ...",
            super::MAX_SWARM_MEMBERS
        ),
        retry_after_secs: None,
    });
    return None;
}
```
`MAX_SWARM_MEMBERS = 1000`, defined in `jcode-swarm-core/src/lib.rs:60` and
re-exported at `swarm.rs:32` (confirmed by the Phase 1 map; I additionally
re-read the doc comment on the re-export directly:
```rust
// crates/jcode-app-core/src/server/swarm.rs:26-32
/// Maximum number of live members (agents) in a single swarm. Re-exported from
/// `jcode_swarm_core` so the server, tools, and prompts all agree on the one
/// runaway-prevention cap for the task-graph model. Normal and light swarms are
/// root-only, one-level fan-out. Deep-swarm roots may create recursive trees with
/// no depth limit, but both the configurable live-worker budget and this absolute
/// cap still apply.
pub(super) use jcode_swarm_core::MAX_SWARM_MEMBERS;
```
) — this doc comment is a strong quotable summary of the whole mode-gate
story in one place.

**(c) The configurable "RAM safety" budget** (a second, independent, lower
cap on top of the hard 1000):
```rust
// crates/jcode-app-core/src/server/comm_session.rs:1363-1378
// `swarm_max_concurrent_agents` is the machine-safety budget shared by
// run_plan and deep recursive spawning. Previously only run_plan obeyed it,
// so nested agents could grow to the 1000-member hard cap and exhaust RAM.
let live_agent_limit = (configured_live_agent_limit > 0)
    .then(|| configured_live_agent_limit.min(super::MAX_SWARM_MEMBERS));
if live_agent_limit.is_some_and(|limit| live_spawned_agent_count >= limit) {
    ...
}
```
Good "why two caps, not one" teaching point: the 1000-member cap is an
absolute safety valve; `swarm_max_concurrent_agents` is a
user/operator-configurable, machine-resource-aware budget layered on top,
and the comment even documents a real historical bug it fixes (deep
recursive spawns previously bypassed the configurable limit and could
exhaust RAM before hitting the hard 1000 cap).

**Live vs. counted-for-cap distinction** (worth a short aside): not every
`SwarmMember` counts against these caps — terminal (finished/stopped/failed)
members don't:
```rust
// crates/jcode-app-core/src/server/swarm.rs:234-236
pub(super) fn member_consumes_swarm_capacity(member: &SwarmMember) -> bool {
    !member_status_is_terminal(&member.status)
}
```

## 5. `report_back_to_session_id` — where the ancestry edge is actually *set* (not just read)

The Phase 1 map documented `swarm_ancestors` (how the chain is *walked*)
thoroughly but did not trace where the field is *written* at spawn time. I
traced it:

- **Visible-spawn path**: `register_visible_spawned_member` constructs the
  brand-new `SwarmMember` directly:
  ```rust
  // crates/jcode-app-core/src/server/comm_session.rs:502-517 (relevant field only)
  SwarmMember {
      session_id: session_id.to_string(),
      ...
      report_back_to_session_id: report_back_to_session_id.map(str::to_string),
      ...
      role: "agent".to_string(),
      ...
  }
  ```
  called from `spawn_swarm_agent` with `Some(req_session_id)` as that
  parameter (comm_session.rs:722, inside the block quoted in §2 step 7) —
  i.e. **the parent edge is always the id of whichever session's `swarm`
  tool call triggered the spawn.**
- **Headless/inline path (the default)**: `create_headless_session` takes
  the same information as an explicit parameter:
  ```rust
  // crates/jcode-app-core/src/server/headless.rs:37-54 (signature only)
  pub(super) async fn create_headless_session(
      ...,
      mcp_pool: Option<Arc<crate::mcp::SharedMcpPool>>,
      report_back_to_session_id: Option<String>,
      memory_scope: HeadlessMemoryScope,
  ) -> Result<String> {
  ```
  and `spawn_swarm_agent` calls it with
  `Some(req_session_id.to_string())` at exactly that parameter position
  (comm_session.rs:678, in the fallback branch quoted in §2 step 4) —
  confirming positionally that headless/inline spawns (the common case,
  since `Inline` is `#[default]`) get the same parent-edge treatment as
  visible ones. I did **not** open the body of `create_headless_session`
  far enough to find its own `SwarmMember`-construction call site, so I
  can't cite an exact line number for where inside `headless.rs` the field
  actually lands in the new member record — only that the correct value is
  demonstrably threaded into the function call. Flagged as an open item if
  a later phase wants that exact line.

One more small but genuinely interesting detail found along the way:
```rust
// crates/jcode-app-core/src/server/headless.rs:17-31
/// Which memory store a headless session gets.
///
/// A bare `bool` here is one typo away from silently reintroducing #729, where
/// every real swarm worker was forced into throwaway test storage and could
/// never read what the session that spawned it remembered. Naming the two cases
/// makes the wrong one hard to pick by accident and obvious in review.
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub(super) enum HeadlessMemoryScope {
    RealProject,
    IsolatedTest,
}
```
Good small aside for the chapter (optional, not essential): a real
regression (`#729`) from using a bare `bool` for a two-state parameter,
fixed by naming the two states — a nice, tiny, concrete "primitive
obsession" lesson with a TS-relevant moral (`boolean` params are exactly as
easy to transpose by accident in TS/JS; the fix — `Memory scope: 'real' |
'isolated'` — is arguably even easier to reach for in TS than in Rust).

## 6. Reparenting on departure — CLOSES MAP GAP #2 (verified, `swarm.rs::remove_session_from_swarm`)

I read the full function (`crates/jcode-app-core/src/server/swarm.rs:995-1219`,
225 lines — confirmed to be the largest function in the file) line by
line. `SWARM_ARCHITECTURE.md:40-45`'s claim is **confirmed accurate**, with
a precise citation, plus additional mechanics the doc doesn't mention.

The doc comment directly above the reparenting logic states the rationale
plainly:
```rust
// crates/jcode-app-core/src/server/swarm.rs:1115-1121
// Reparent the departing member's direct children so the spawn tree never
// holds dangling report-back edges. Orphaned subtrees would otherwise
// silently change ownership semantics: stop permissions, subtree broadcast
// scope, and completion report-back all walk this chain. Children are
// attached to their grandparent when it is still a live member of this
// swarm, otherwise to the current coordinator, otherwise they become
// roots (report_back_to_session_id = None).
```
The fallback-parent computation:
```rust
// crates/jcode-app-core/src/server/swarm.rs:1122-1142
let fallback_parent: Option<String> = {
    let grandparent_is_live = if let Some(ref parent) = departing_parent {
        parent != session_id && {
            let members = swarm_members.read().await;
            members.get(parent).is_some_and(|member| member.swarm_id.as_deref() == Some(swarm_id))
        }
    } else {
        false
    };
    if grandparent_is_live {
        departing_parent.clone()
    } else {
        let coordinators = swarm_coordinators.read().await;
        coordinators.get(swarm_id).filter(|coordinator| coordinator.as_str() != session_id).cloned()
    }
};
```
And the actual reparenting loop:
```rust
// crates/jcode-app-core/src/server/swarm.rs:1143-1156
let mut reparented: Vec<String> = Vec::new();
{
    let mut members = swarm_members.write().await;
    for member in members.values_mut() {
        if member.swarm_id.as_deref() == Some(swarm_id)
            && member.report_back_to_session_id.as_deref() == Some(session_id)
        {
            member.report_back_to_session_id = fallback_parent
                .clone()
                .filter(|parent| parent != &member.session_id);
            reparented.push(member.session_id.clone());
        }
    }
}
```
So, precisely: **grandparent if it's still a live member of the same
swarm → else the current coordinator (excluding the departing session
itself) → else `None` (promoted to root)**, exactly matching
`SWARM_ARCHITECTURE.md:40-45`'s prose ("they attach to their live
grandparent, falling back to the current coordinator, else they become
roots"). The `.filter(|parent| parent != &member.session_id)` guard on the
last line prevents a self-loop edge case (a child becoming its own parent).

**Bonus mechanic the design doc doesn't spell out, worth including**: if
the *departing* member was itself the swarm's coordinator, a **new
coordinator is elected** before reparenting happens — lowest session id
among live, non-headless members:
```rust
// crates/jcode-app-core/src/server/swarm.rs:1054-1069 (selection)
let new_coordinator = {
    let swarms = swarms_by_id.read().await;
    let members = swarm_members.read().await;
    swarms.get(swarm_id).and_then(|swarm| {
        swarm.iter().filter_map(|id| {
            members.get(id).filter(|member| !member.is_headless).map(|_| id.clone())
        }).min()
    })
};
```
(Full succession handling: lines 1046-1106 — role update, plan-participant
registration, and a `Notification` sent to the new coordinator's own
`event_tx`, "You are now the coordinator for this swarm.") This is a
genuinely nice small companion exhibit to the reparenting logic: departure
handling has to fix up *two* independent kinds of "who's in charge" state
(the coordinator slot for the whole swarm, and the parent pointer for each
individual departing member's children) in the same teardown function.

**TS-equivalent mapping** for this whole section: reparenting-on-departure
is structurally identical to removing a node from an in-memory tree keyed
by parent-pointers stored on the children (not by child-lists on the
parent) — i.e. a `Map<childId, parentId>` where removing an entry requires
scanning for every child whose `parentId === removedId` and rewriting it.
Coordinator election-on-departure is a `Math.min` over live candidate ids,
directly portable.

## 7. Summary list of exact snippets to feature (file:line, all verified this pass)

| # | What | Citation | TS-analogue note |
|---|------|----------|-------------------|
| 1 | `swarm` tool schema + action enum | `crates/jcode-app-core/src/tool/communicate.rs:1939-1963` | a typed action-dispatch object, e.g. Zod/JSON-schema-validated `{action, ...}` payload |
| 2 | `spawn` action required-field `anyOf` | `communicate.rs:2154-2187` | discriminated union validation (per-variant required fields) |
| 3 | `spawn` action dispatch → `Request::CommSpawn` | `communicate.rs:2720-2751` | `dispatch(action)` to a single reducer/owner over a channel |
| 4 | `spawn_swarm_agent` full walkthrough | `crates/jcode-app-core/src/server/comm_session.rs:557-827` | a `spawnWorker(parentId, opts)` function returning a new id |
| 5 | Visible-vs-headless branch | `comm_session.rs:622-652` | choosing `child_process.spawn` (visible) vs. an in-process async task (headless) |
| 6 | Fire-and-forget initial-turn task | `comm_session.rs:772-822` | calling an `async function` without `await` — a floating promise |
| 7 | `SwarmSpawnMode` enum | `crates/jcode-config-types/src/lib.rs:643-657` | a string union type `'visible'\|'headless'\|'inline'\|'auto'` |
| 8 | Mode-gate (root's effort, not child's) | `comm_session.rs:1330-1348` | authorize against a capability derived from the root of a call chain |
| 9 | Hard member cap (1000) | `comm_session.rs:1350-1361`, const at `crates/jcode-app-core/src/server/swarm.rs:26-32` | a fixed-size worker-pool ceiling |
| 10 | Configurable live-agent budget | `comm_session.rs:1363-1378` | a second, operator-tunable concurrency limit layered on the hard cap |
| 11 | `swarm_ancestors` (chain walk) | `swarm.rs:34-60` | walking a `Map<childId, parentId>` upward, cycle-guarded with a visited `Set` |
| 12 | Ancestry edge set at spawn time | `comm_session.rs:502-517` (visible) + `headless.rs:37-54`/`comm_session.rs:678` (headless, positional proof) | `parentMap.set(newId, callerId)` |
| 13 | Reparenting-on-departure | `swarm.rs:1115-1156` | rewriting every `Map` entry whose value equals the removed id |
| 14 | Coordinator election-on-departure | `swarm.rs:1046-1106` | `Math.min` over live candidate ids |

## 8. Open questions for later phases

1. Exact `SwarmMember`-construction line inside `headless.rs` for the
   headless/inline spawn path is not pinned down (positional-parameter
   proof only, see §5). Open before quoting a specific line number for it.
2. The "ad hoc / light / deep" naming is my own synthesis from the effort
   sentinels (`prompt.rs:110-131`) and the gate (`comm_session.rs:1330-1348`)
   — there is no single Rust `enum` with exactly these three variant names.
   The handbook must present this as "three effort-keyed operating modes,"
   not quote a nonexistent `enum Mode { AdHoc, Light, Deep }`.
3. `run_swarm_task`/`run_swarm_message`'s real reachability (debug-socket
   only, §0 above) is a significant correction to the original plan's
   framing and should be flagged again explicitly whenever Chapter 3 is
   written, even though that's outside this chapter's scope.
4. I did not verify whether `dag::Mode` (`Deep`/`Light`, in
   `crates/jcode-plan/src/dag/mod.rs:36-49`, per the Phase 1 map) is
   literally *the same* enum consulted anywhere in the `comm_session.rs`
   mode-gate I read, or a fully separate mode concept used only inside the
   `task_graph`/`run_plan` machinery. My reading of `ensure_spawn_coordinator_swarm`
   shows it consults `session_effort`/`is_deep_swarm_effort` (string-based),
   not `dag::Mode` directly — worth a later phase confirming whether/how
   the two are bridged (e.g. does seeding a `task_graph` with `mode:"deep"`
   get cross-checked against the seeding session's effort anywhere?) before
   asserting a definite relationship in prose.
