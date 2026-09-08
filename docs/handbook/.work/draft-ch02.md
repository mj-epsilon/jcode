# Chapter 2 — Anatomy of a Spawn

## Two paths, and only one of them is real

Before we trace anything, a correction that matters more than it might
seem. Open `swarm.rs` and you'll find a function pair, `run_swarm_task` /
`run_swarm_message`, built around `try_join_all` — the kind of
planner-fans-out-and-awaits-everything shape that looks exactly like "the"
spawn mechanism a design doc would draw a diagram of.

It is real, working code. It is also **not what happens when a live agent
calls the swarm tool.** Tracing every caller of `run_swarm_message` turns
up exactly two, and both are debug-socket string commands, not tool calls:

```rust
// jcode: crates/jcode-app-core/src/server/debug_command_exec.rs:132-139
if trimmed.starts_with("swarm_message:") {
    let msg = trimmed.strip_prefix("swarm_message:").unwrap_or("").trim();
    // ...
    let final_text = super::run_swarm_message(agent.clone(), msg).await?;
    return Ok(final_text);
}
```

```rust
// jcode: crates/jcode-app-core/src/server/debug_jobs.rs:77-95
if trimmed.starts_with("swarm_message_async:") {
    // ...
    let result = super::run_swarm_message(agent.clone(), &msg).await;
```

Meanwhile the `swarm` tool's own `"message"` action — the thing an agent
actually calls — does something else entirely: it routes a direct message,
a channel post, or a broadcast. It never calls `run_swarm_message`. So:
**`run_swarm_task`/`run_swarm_message` is debug/dev tooling** — useful for
testing the orchestration engine from a socket, but not production
behavior a spawned agent can trigger. We won't build this chapter's mental
model on it.

The real, agent-facing path — the one this whole chapter is about — is:

```
swarm tool, action: "spawn"
  → Request::CommSpawn
  → handle_comm_spawn (server-side)
  → spawn_swarm_agent
```

`spawn_swarm_agent` lives at
`crates/jcode-app-core/src/server/comm_session.rs:557-827`, and it's the
one function to actually understand in this chapter.

## Step 1: the tool call itself

`swarm` is a single, multi-action tool — one schema, many `action` values.
Its name is asserted directly in the `Tool` trait implementation:

```rust
// jcode: crates/jcode-app-core/src/tool/communicate.rs:1939-1946
impl Tool for CommunicateTool {
    fn name(&self) -> &str {
        "swarm"
    }
```

The full action list runs to nearly forty values (`crates/jcode-app-core/
src/tool/communicate.rs:1954-1963`) — plans, channels, the task graph, and
more, each belonging to a later chapter. `"spawn"` is one value among them.
The fields that matter for spawning:

```rust
// jcode: crates/jcode-app-core/src/tool/communicate.rs:2009-2025, 2043-2047
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
// ...
"spawn_mode": {
    "type": "string",
    "enum": ["visible", "headless", "inline", "auto"],
    "description": "Spawn UI mode: visible terminal, headless, inline gallery, or auto. Defaults to inline."
},
```

`label` is required only when `action: "spawn"` — enforced with a JSON
Schema `anyOf` instead of a flat top-level `required` list, specifically so
the requirement doesn't leak onto every other action:

```rust
// jcode: crates/jcode-app-core/src/tool/communicate.rs:2154-2187
// `swarm` is a multi-action tool, so putting `label` in the top-level
// `required` array would incorrectly require it for read/list/message and
// every other action. Use mutually exclusive action branches instead...
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

That inline comment is worth pausing on: a real, dated cross-provider bug —
Gemini's schema validator rejects a `required` field it can't find declared
locally — shaped this exact bit of JSON Schema. Multi-model tool schemas
have their own cross-browser-style portability quirks.

The `spawn` action handler itself doesn't touch any swarm state directly:

```rust
// jcode: crates/jcode-app-core/src/tool/communicate.rs:2720-2751
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
        // ...
    }
}
```

It builds a typed `Request` value and awaits a response over an internal
channel. The code that parses the LLM's tool-call JSON never reaches into a
shared registry directly — it hands off a message and waits.

```typescript
// idiomatic TS equivalent — this code does not exist in jcode
type SpawnRequest = {
  kind: "spawn";
  sessionId: string;
  workingDir?: string;
  initialMessage?: string;
  spawnMode?: "visible" | "headless" | "inline" | "auto";
  label: string;
};

// The tool handler never touches swarm state directly. It posts a
// typed action and awaits a typed reply — like dispatching to a
// Redux-style reducer, or postMessage()-ing a dedicated worker that
// owns the real state.
async function handleSpawnAction(params: { label?: string; workingDir?: string; prompt?: string }) {
  if (!params.label) throw new Error("label is required for spawn");
  const request: SpawnRequest = {
    kind: "spawn",
    sessionId: currentSessionId(),
    workingDir: params.workingDir,
    initialMessage: params.prompt,
    label: params.label,
  };
  const response = await sendRequest(request);
  if (response.kind === "spawnResponse" && response.newSessionId) {
    return `Spawned new agent: ${response.newSessionId}`;
  }
  throw new Error("spawn failed");
}
```

## Step 2: `spawn_swarm_agent`, the real work

This is the function the tool's request ultimately reaches —
`crates/jcode-app-core/src/server/comm_session.rs:557-827`, 271 lines. Its
signature:

```rust
// jcode: crates/jcode-app-core/src/server/comm_session.rs:557-579
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
    // ... 12 more shared-state/channel params
) -> anyhow::Result<String> {
```

Walking through it, in order:

1. **Resolve the working directory and auth identity to inherit** — a
   spawned child inherits the parent's exact provider/model/auth route
   rather than falling back to config defaults.
2. **Resolve the spawn's UI mode**, falling back to the configured default
   when the caller didn't specify one (`let resolved_spawn_mode =
   spawn_mode.unwrap_or(agents_config.swarm_spawn_mode);`,
   `comm_session.rs:591`).
3. **Prepare the startup message** — if the caller gave an
   `initial_message`, it's appended with instructions telling the new
   worker to call `swarm report` before finishing, so the parent has
   something to read back later.
4. **Try a visible spawn; fall back to headless** — the mode gate covered
   in detail below.
5. **Register the new member with the swarm's plan**, but only if the plan
   already has items or participants, and **broadcast the plan update**.
6. **Register the `SwarmMember` bookkeeping record** — only on the
   non-headless-fallback branch (more on this below), then **set the task
   label**, then **persist state**, before anything asynchronous fires.
7. **Conditionally kick off the first turn as a fire-and-forget task** —
   covered in its own section below.

```typescript
// idiomatic TS equivalent — this code does not exist in jcode
async function spawnWorker(parentId: string, opts: SpawnOptions): Promise<string> {
  const workingDir = resolveSpawnWorkingDir(opts.workingDir);
  const identity = resolveInheritedIdentity(parentId);
  const spawnMode = opts.spawnMode ?? config.defaultSpawnMode;

  const startupMessage = opts.initialMessage
    ? appendCompletionReportInstructions(opts.initialMessage)
    : undefined;

  const { newSessionId, isHeadlessFallback } = await attemptSpawn(spawnMode, {
    workingDir, identity, startupMessage,
  });

  if (planHasItemsOrParticipants(swarmId)) {
    registerPlanParticipant(swarmId, newSessionId);
    broadcastPlanUpdate(swarmId);
  }

  if (!isHeadlessFallback) {
    registerMember({ sessionId: newSessionId, swarmId, reportBackTo: parentId });
  }
  setMemberTaskLabel(newSessionId, opts.label ?? opts.initialMessage);
  persist();

  if (isHeadlessFallback && startupMessage) {
    void runFirstTurn(newSessionId, startupMessage); // fire-and-forget, see below
  }
  return newSessionId;
}
```

### The visible-vs-headless branch

```rust
// jcode: crates/jcode-app-core/src/server/comm_session.rs:622-652
let visible_spawn = match resolved_spawn_mode {
    // Inline workers run in-process like headless ones; the difference is
    // purely how the coordinator renders them (a live inline gallery).
    SwarmSpawnMode::Headless | SwarmSpawnMode::Inline => {
        Err(anyhow::anyhow!("headless spawn requested"))
    }
    SwarmSpawnMode::Visible | SwarmSpawnMode::Auto => prepare_visible_spawn_session(...),
};
```

The `Err(...)` here for `Headless`/`Inline` isn't a real failure — it's a
control-flow shortcut into the same fallback branch that a genuine
visible-spawn failure would also take:

```rust
// jcode: crates/jcode-app-core/src/server/comm_session.rs:654-695
let (new_session_id, is_headless_fallback) = match visible_spawn {
    Ok((new_session_id, true)) => Ok((new_session_id, false)),
    Ok((_, false)) | Err(_) => {
        // ...
        create_headless_session(..., Some(req_session_id.to_string()), ...).await
    }
}?;
```

## Step 3: two different things people call "mode"

Here's a genuine gotcha in the source: jcode has **two separate concepts**
both casually called "mode," and mixing them up will make the rest of this
chapter confusing.

### Mode axis #1 — `SwarmSpawnMode`: how the worker's UI is rendered

```rust
// jcode: crates/jcode-config-types/src/lib.rs:643-657
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

The important plain-English point: **`Headless` and `Inline` both mean "the
child runs as an async task inside the same OS process as the parent
server."** No new process, no new OS thread — just another concurrently
running job. Only `Visible` (and `Auto`'s attempt at it) opens a genuinely
separate terminal/process. And since `Inline` is the `#[default]`, that's
the common case in practice.

```typescript
// idiomatic TS equivalent — this code does not exist in jcode
type SwarmSpawnMode = "visible" | "headless" | "inline" | "auto";

function attemptSpawn(mode: SwarmSpawnMode, opts: SpawnOpts) {
  if (mode === "headless" || mode === "inline") {
    // headless/inline ~ another async job in THIS process,
    // tracked in a Map<sessionId, Promise<void>> — not a worker_thread.
    return runInProcess(opts);
  }
  // visible/auto ~ actually spawning a subprocess/window —
  // child_process.spawn(...), a meaningfully different primitive.
  return spawnVisibleWindow(opts).catch(() => runInProcess(opts));
}
```

### Mode axis #2 — "ad hoc / light / deep": how much recursive spawning is allowed

This is the axis this chapter's title is actually about, and it's **not** a
Rust `enum` at all — there's no `enum Mode { AdHoc, Light, Deep }` anywhere
in the codebase. It's three operating modes distinguished by a session's
recorded *reasoning effort string*:

```rust
// jcode: crates/jcode-base/src/prompt.rs:129-131
pub fn is_deep_swarm_effort(effort: &str) -> bool {
    effort.trim().eq_ignore_ascii_case(SWARM_DEEP_EFFORT)
}
```

`SWARM_DEEP_EFFORT` is the literal string `"swarm-deep"`. Its sibling
`"swarm"` (`SWARM_EFFORT`) is the **light** rung — swarm orchestration
enabled, but without the recursive DAG machinery deep mode unlocks. **Ad
hoc** is simply neither sentinel: ordinary spawning, still allowed by
default, since a first generation of spawns isn't gated by effort at all —
only *recursive* (grandchild) spawning is.

A session's current effort is tracked in a small side-table, deliberately
kept separate from the agent's own lock to avoid a deadlock:

```rust
// jcode: crates/jcode-app-core/src/session_effort.rs:1-19
//! The swarm task-graph seed handler runs on the server socket thread while the
//! *seeding* agent is blocked inside its `swarm` tool call holding its own agent
//! lock. That means the handler cannot read the seeder's effort via the agent
//! mutex without deadlocking. This tiny side-table is updated whenever an agent's
//! effort changes (cheap string writes) so server handlers can learn a session's
//! effort by id without taking any agent lock.
```

That's a nice, concrete "why a side-index instead of the obvious lookup"
example: the obvious path (ask the agent object for its own effort) would
deadlock, because the socket handler needs the answer *while* the agent is
still holding its own lock inside the very tool call that triggered the
question. A cheap, separately-locked side-table sidesteps that entirely.

### The actual gate

The mode-gate and the member cap both live inside one function,
`ensure_spawn_coordinator_swarm`, `comm_session.rs:1234-1431` — 198 lines,
three independent checks.

**Check (a): only the root may spawn recursively, and it's the *root's*
effort that's checked, not the caller's own.**

```rust
// jcode: crates/jcode-app-core/src/server/comm_session.rs:1330-1348
// Light and ad hoc swarms are deliberately one-level fan-out: only the root
// session may create workers. Recursive spawning is an explicit deep-swarm
// capability, keyed from the root's effort rather than the requesting
// child's effort so a worker cannot opt itself into unbounded growth.
if !is_root {
    let root_is_deep = crate::session_effort::session_effort(&root_session_id)
        .as_deref()
        .is_some_and(crate::prompt::is_deep_swarm_effort);
    if !root_is_deep {
        // reject: "Recursive swarm spawning is disabled for light and ad hoc swarms..."
        return None;
    }
}
```

This is a genuinely good access-control pattern, worth naming plainly: the
check authorizes against a capability derived from the **root of the call
chain**, not a value the requester itself controls. A worker can't declare
"I'm running at deep effort" to unlock its own recursive spawning — only
the root's recorded state counts.

```typescript
// idiomatic TS equivalent — this code does not exist in jcode
function canSpawnRecursively(requester: SwarmMember, rootSessionId: string): boolean {
  if (requester.isRoot) return true; // roots always get a first generation
  // Authorize against the ROOT's capability, never the caller's own claim —
  // a worker cannot self-elevate by lying about its own effort.
  return isDeepSwarmEffort(sessionEffort(rootSessionId));
}
```

**Check (b): an absolute hard cap**, 1000 live members, enforced even if the
configurable limit below is turned off:

```rust
// jcode: crates/jcode-app-core/src/server/comm_session.rs:1350-1361
if live_member_count >= super::MAX_SWARM_MEMBERS {
    // reject: "Swarm member limit reached (hard max {MAX_SWARM_MEMBERS})..."
    return None;
}
```

```rust
// jcode: crates/jcode-app-core/src/server/swarm.rs:26-32
/// Maximum number of live members (agents) in a single swarm. Re-exported from
/// `jcode_swarm_core` so the server, tools, and prompts all agree on the one
/// runaway-prevention cap for the task-graph model. Normal and light swarms are
/// root-only, one-level fan-out. Deep-swarm roots may create recursive trees with
/// no depth limit, but both the configurable live-worker budget and this absolute
/// cap still apply.
pub(super) use jcode_swarm_core::MAX_SWARM_MEMBERS;
```

**Check (c): a second, configurable, lower budget** on top of the hard cap
— an operator-tunable "don't exhaust the machine's RAM" limit that used to
apply only to one code path and got widened after a real bug:

```rust
// jcode: crates/jcode-app-core/src/server/comm_session.rs:1363-1378
// `swarm_max_concurrent_agents` is the machine-safety budget shared by
// run_plan and deep recursive spawning. Previously only run_plan obeyed it,
// so nested agents could grow to the 1000-member hard cap and exhaust RAM.
let live_agent_limit = (configured_live_agent_limit > 0)
    .then(|| configured_live_agent_limit.min(super::MAX_SWARM_MEMBERS));
```

Two caps, two purposes: 1000 is an absolute safety valve; the configurable
budget is a smaller, operator-set ceiling for machines that can't run a
thousand concurrent agents. Not every counted member stays counted forever
— finished, stopped, or failed members stop consuming capacity:

```rust
// jcode: crates/jcode-app-core/src/server/swarm.rs:234-236
pub(super) fn member_consumes_swarm_capacity(member: &SwarmMember) -> bool {
    !member_status_is_terminal(&member.status)
}
```

```typescript
// idiomatic TS equivalent — this code does not exist in jcode
const MAX_SWARM_MEMBERS = 1000; // absolute hard ceiling, never configurable

function canSpawnAnother(swarm: SwarmState, configuredLimit: number): boolean {
  const liveCount = [...swarm.members.values()]
    .filter(m => !isTerminalStatus(m.status)).length;
  if (liveCount >= MAX_SWARM_MEMBERS) return false;
  const softLimit = configuredLimit > 0 ? Math.min(configuredLimit, MAX_SWARM_MEMBERS) : Infinity;
  return liveCount < softLimit;
}
```

## The fire-and-forget first turn

When a headless/inline spawn includes a startup message, `spawn_swarm_agent`
kicks off the first turn without waiting for it:

```rust
// jcode: crates/jcode-app-core/src/server/comm_session.rs:772-822
tokio::spawn(async move {
    update_member_status(&sid_clone, "running", ..., ...).await;
    let event_tx = super::session_event_fanout_sender(...);
    let start_message_index = { let agent = agent_arc.lock().await; agent.message_count() };
    let result = process_message_streaming_mpsc(Arc::clone(&agent_arc), &initial_msg, vec![], None, event_tx).await;
    let completion_report = if result.is_ok() { /* ... */ } else { None };
    let (new_status, new_detail) = match result {
        Ok(()) => ("ready", None),
        Err(ref error) => ("failed", Some(truncate_detail(&error.to_string(), 120))),
    };
    update_member_status_with_report(&sid_clone, new_status, new_detail, completion_report, ...).await;
});
```

`tokio::spawn(async move { ... })` starts a task running concurrently and
hands back a `JoinHandle` — which is immediately dropped here, never
`.await`ed. That's exactly what "fire-and-forget" means in async Rust: the
outer function returns the new session id right away, without waiting to
see how the first turn goes. The only way anything downstream learns how it
went is by later reading the member's status or completion report — never
through a `Result` handed back to whoever triggered the spawn.

The direct TypeScript analogue is one every linter warns about for a
reason: calling an `async function` without `await`ing it — a **floating
promise**. If it rejects and nothing is listening, the error vanishes
silently unless a fallback is wired up (jcode's status/report fields, here).

```typescript
// idiomatic TS equivalent — this code does not exist in jcode
function fireFirstTurn(sessionId: string, initialMessage: string) {
  updateMemberStatus(sessionId, "running");
  // No `await` here — deliberately fire-and-forget. Most TS linters
  // (no-floating-promises) would flag this exact line by default,
  // because errors below silently disappear unless handled here.
  runFirstTurn(sessionId, initialMessage)
    .then(() => updateMemberStatusWithReport(sessionId, "ready", null, latestReport(sessionId)))
    .catch(err => updateMemberStatusWithReport(sessionId, "failed", String(err).slice(0, 120), null));
  // spawnWorker() already returned sessionId before this ever settles.
}
```

This is a deliberate tradeoff, not an oversight: whoever spawned this
worker gets an id back immediately and has to poll or subscribe for status,
because there's no synchronous "did it work" signal.

## Ancestry: where `report_back_to_session_id` actually gets set

Chapter 1 introduced `report_back_to_session_id` as the one field that lets
the whole spawn tree be derived instead of stored. Here's where it's
actually written.

For a visible spawn, the new `SwarmMember` record is built directly:

```rust
// jcode: crates/jcode-app-core/src/server/comm_session.rs:502-519 (relevant field only)
SwarmMember {
    session_id: session_id.to_string(),
    // ...
    report_back_to_session_id: report_back_to_session_id.map(str::to_string),
    // ...
    role: "agent".to_string(),
    // ...
}
```

called with `Some(req_session_id)` — the id of whichever session's `swarm`
tool call triggered the spawn.

For the default, headless/inline path, the same value is threaded through
as a parameter to `create_headless_session`:

```rust
// jcode: crates/jcode-app-core/src/server/headless.rs:37-54 (signature only)
pub(super) async fn create_headless_session(
    // ...
    report_back_to_session_id: Option<String>,
    memory_scope: HeadlessMemoryScope,
) -> Result<String> {
```

called with `Some(req_session_id.to_string())` at that exact parameter
position. That confirms the parent edge gets threaded into the headless
path correctly too — though the exact line inside `headless.rs` where it
lands in the constructed `SwarmMember` wasn't traced this pass, so it's
worth treating as "proven by the call site, not pinned to a specific line
inside that function."

```typescript
// idiomatic TS equivalent — this code does not exist in jcode
function registerMember(sessionId: string, swarmId: string, reportBackTo: string | null) {
  members.set(sessionId, {
    sessionId,
    swarmId,
    reportBackToSessionId: reportBackTo, // parentMap.set(newId, callerId)
    role: "agent",
    status: "starting",
  });
}
```

## Reparenting when a member leaves

If jcode only ever added members, the parent-pointer design from Chapter 1
would be trivial. The interesting part is what happens when a member with
children of its own **departs** the swarm — those children can't be left
pointing at a session that no longer exists. `remove_session_from_swarm`
(`swarm.rs:995-1213`, the largest function in the file) handles it, and its
own doc comment states the stakes plainly:

```rust
// jcode: crates/jcode-app-core/src/server/swarm.rs:1115-1121
// Reparent the departing member's direct children so the spawn tree never
// holds dangling report-back edges. Orphaned subtrees would otherwise
// silently change ownership semantics: stop permissions, subtree broadcast
// scope, and completion report-back all walk this chain. Children are
// attached to their grandparent when it is still a live member of this
// swarm, otherwise to the current coordinator, otherwise they become
// roots (report_back_to_session_id = None).
```

The fallback chain, computed directly:

```rust
// jcode: crates/jcode-app-core/src/server/swarm.rs:1122-1142
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

And the loop that actually rewrites every affected child:

```rust
// jcode: crates/jcode-app-core/src/server/swarm.rs:1143-1156
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

Precisely: **grandparent, if it's still a live member of the same swarm →
else the current coordinator (excluding the departing session itself) →
else `None`, promoted to root.** The trailing `.filter(...)` guards against
a self-loop, where a member could otherwise end up listed as its own
parent.

One more piece the design docs don't mention: if the *departing* member was
itself the coordinator, a new one is elected — the lowest session id among
the swarm's remaining live, non-headless members — before reparenting even
happens:

```rust
// jcode: crates/jcode-app-core/src/server/swarm.rs:1056-1070
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

So departure teardown has to fix up two independent kinds of "who's in
charge" in one pass: the coordinator slot for the whole swarm, and the
parent pointer for each of the departing member's own children.

```typescript
// idiomatic TS equivalent — this code does not exist in jcode
function removeMember(sessionId: string, swarm: SwarmState) {
  // 1. If the departing member was coordinator, elect a new one first —
  //    lowest session id among live, non-headless members.
  if (swarm.coordinators.get(swarm.id) === sessionId) {
    const next = [...swarm.members.values()]
      .filter(m => !m.isHeadless && m.sessionId !== sessionId)
      .map(m => m.sessionId).sort()[0];
    if (next) {
      swarm.coordinators.set(swarm.id, next);
      swarm.members.get(next)!.role = "coordinator";
    }
  }

  // 2. Reparent every direct child of the departing member: grandparent if
  //    still live, else the coordinator, else promote to root.
  const parent = swarm.members.get(sessionId)?.reportBackToSessionId ?? null;
  const grandparentLive = parent !== null && parent !== sessionId
    && swarm.members.get(parent)?.swarmId === swarm.id;
  const fallback = grandparentLive ? parent
    : (swarm.coordinators.get(swarm.id) !== sessionId ? swarm.coordinators.get(swarm.id) ?? null : null);

  for (const member of swarm.members.values()) {
    if (member.swarmId === swarm.id && member.reportBackToSessionId === sessionId) {
      member.reportBackToSessionId = fallback === member.sessionId ? null : fallback;
    }
  }
  swarm.members.delete(sessionId);
}
```

Structurally, this whole section is "removing a node from a tree keyed by
parent-pointers on the children, not child-lists on the parent" — removal
means scanning for every node whose parent pointer equals the removed id
and rewriting it. Coordinator election is a plain `min`/`sort` over live
candidate ids. Neither needs anything Rust-specific to understand.

## What this chapter didn't cover

Two things worth flagging rather than glossing over: the exact line inside
`headless.rs` where the `SwarmMember` record gets constructed for
headless/inline spawns wasn't traced this pass — only that the parent id is
correctly passed into `create_headless_session`. And the relationship
between the effort-string mode gate covered here and a separate
`Deep`/`Light` mode enum used by the task-graph machinery (Chapter 6
territory) hasn't been confirmed — they may or may not be cross-checked
against each other. Treat them as two different concepts until proven
otherwise.

---

**Next**: Chapter 3 covers what happens *after* a spawn — the three
distinct fan-out/fan-in patterns jcode uses once multiple agents are
actually running concurrently.
