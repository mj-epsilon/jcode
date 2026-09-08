# Chapter 1 — Why Parallel Agents, and Why This Codebase

## One agent, one thread, one bottleneck

A coding agent, at its simplest, is a loop: read something, think about it,
take one action, look at what happened, repeat. One tool call at a time. One
working directory. One conversation thread carrying all the context.

That loop is a fine model for a scoped task — "fix this one function,"
"explain this error." It falls apart the moment the work is wide instead of
deep. "Explore this entire codebase." "Fix these forty independent lint
violations." "Implement six unrelated features before end of day." None of
those sub-tasks depend on each other, and yet a single serial agent will
still do them one after another, paying the full cost of each one before
starting the next.

This is the same realization that gave us parallel CI: if your test suite
takes an hour and none of the tests depend on each other, you don't make the
suite faster by writing quicker tests — you shard it across workers and run
it in twenty minutes. The jcode swarm system applies that idea to coding
agents. The hard part was never "how do I start more than one agent" —
`tokio::spawn` on its own can do that in one line. The hard part is: how do
you let one agent become many, all working at once, without turning shared
state into a race condition and without losing track of who is doing what
for whom.

This chapter builds the vocabulary you'll need for the rest of the book, and
is honest about one thing up front: not every term you'll see in jcode's own
design docs corresponds to a real, load-bearing piece of code. Part of
learning this system is learning to tell the difference.

## Why jcode is a good codebase to learn this from

jcode's swarm layer is not built on an actor framework, a job queue, or any
kind of distributed-systems library. There's no `actix`, no `ractor`, no
message broker. It's hand-rolled directly on top of Tokio's own primitives —
`JoinSet`, `CancellationToken`, `broadcast` channels, `RwLock`, `Mutex`, and
plain `tokio::spawn`. This was confirmed by reading the import lists at the
top of the two files that do most of the orchestration work:

```rust
// jcode: crates/jcode-app-core/src/server/swarm.rs:1-20
use futures::future::try_join_all;   // line 9
use tokio::sync::{Mutex, RwLock, broadcast};  // line 20
```

No actor-model crate appears anywhere in the swarm code path. That's good
news for a reader who doesn't know Rust: every concurrency primitive jcode
uses here is a *named, ordinary* building block, not framework-specific
magic. Each one has a direct, nameable equivalent in TypeScript/Node —
`Promise.all`, `worker_threads`, `EventEmitter`, `AbortController` — which
is exactly the mapping this handbook is going to draw, chapter by chapter.

## Vocabulary

### Session, and Agent

A **session** is the persistent identity and conversation state of one
running agent — think "the row in the database that says this conversation
exists and here's its history." An **agent** is the live, in-memory object
that actually drives the loop — it holds the session and does the work.
Spawning a new member of a swarm means creating a brand-new session/agent
pair, literally:

```rust
// jcode: crates/jcode-app-core/src/server/swarm.rs:1548-1551
let mut session = Session::create(
    Some(session_id),
    Some(format!("{} (@{} swarm)", description, subagent_type)),
);
```

*(This particular snippet lives inside a debug-only code path — more on
that caveat in Chapter 2 — but the `Session::create` call shape is
representative of what "creating a session" looks like everywhere in the
codebase.)*

`Session::create` just returns an ordinary struct value — nothing registers
itself anywhere by calling this. The caller decides what to do with it next.

```typescript
// idiomatic TS equivalent — this code does not exist in jcode
class Session {
  static create(id: string | null, title: string | null): Session {
    return new Session(id ?? crypto.randomUUID(), title);
  }
  private constructor(public id: string, public title: string | null) {}
}

// Calling Session.create() does nothing but hand you a value —
// exactly like calling `new Session(...)` directly. No registry,
// no side effects, until *you* do something with it.
const session = Session.create(null, "api reviewer (@agent swarm)");
```

There's a third term worth knowing now, even though its full mechanics
belong to Chapter 2: a **`SwarmMember`** is the live bookkeeping record for
"a session that is part of a swarm." It's a separate struct from `Session`
— tracking things like status, which swarm it belongs to, and who it
reports to — constructed at the point a spawn actually registers:

```rust
// jcode: crates/jcode-app-core/src/server/comm_session.rs:502-528
// (register_visible_spawned_member — constructs the SwarmMember record)
```

So, three related but distinct things: **agent** is the live object doing
the work; **session** is its persisted identity; **`SwarmMember`** is the
swarm's own bookkeeping entry about that session. jcode's design docs
sometimes use "agent" loosely to mean all three — worth untangling early so
later chapters don't feel sloppy about it.

### Coordinator

A swarm can have a **coordinator** — a role, not a special kind of agent.
It's literally a string, and the tool schema that lets an agent claim it
only accepts two values:

```rust
// jcode: crates/jcode-app-core/src/tool/communicate.rs:2005-2008
"role": { "type": "string", "enum": ["agent", "coordinator"] }
```

At most one member per swarm holds the coordinator role at a time, and its
job is narrow: it's the one member allowed to own the swarm's single shared
task plan (propose it, approve it, hand out assignments). Only a **root**
session — one that wasn't itself spawned by anyone — is allowed to claim
the slot, and only when it's empty or the current holder looks stale (every
one of its event channels closed, even without a clean shutdown — a
deliberate guard against a dead coordinator wedging the slot forever).

It's worth stressing what "coordinator" is *not*: it isn't "the agent that
owns this piece of work" or "the parent of this subtree." That relationship
is tracked separately (see `report_back_to_session_id`, below). A member can
be a plain "agent" and still be the parent of five other agents; the
coordinator role only ever concerns who owns the plan.

```typescript
// idiomatic TS equivalent — this code does not exist in jcode
type Role = "agent" | "coordinator";

interface SwarmMember {
  sessionId: string;
  role: Role;
  isRoot: boolean; // no report_back_to_session_id
}

function tryClaimCoordinator(
  swarmId: string,
  candidate: SwarmMember,
  coordinators: Map<string, string>,     // swarmId -> sessionId
  members: Map<string, SwarmMember>,
): boolean {
  if (!candidate.isRoot) return false;
  const current = coordinators.get(swarmId);
  const currentIsStale = current ? isStale(members.get(current)) : true;
  if (current && !currentIsStale) return false;
  coordinators.set(swarmId, candidate.sessionId);
  return true;
}
```

### Swarm

A **swarm** isn't a struct with its own lifecycle at all — it's just a
`swarm_id` string that a set of `SwarmMember`s happen to share. The grouping
itself is one more index sitting in shared state: `swarm_id -> set of
member session ids`.

Where does that id come from? Normally it's derived from the repo's git
common directory, so that multiple `git worktree` checkouts of the same
repository land in the same swarm:

```rust
// jcode: crates/jcode-app-core/src/server/util.rs:329-341
pub(crate) fn swarm_id_for_dir(dir: Option<PathBuf>) -> Option<String> {
    if let Ok(sw_id) = std::env::var("JCODE_SWARM_ID") { /* explicit override */ }
    let dir = dir?;
    if let Some(git_common) = git_common_dir_for(&dir) {
        return Some(git_common.to_string_lossy().to_string());
    }
    Some(dir.to_string_lossy().to_string())
}
```

There's a nuance, though: root sessions actually default to *owning their
own swarm*, rather than automatically sharing one with every other session
opened in the same repo. The code's own comment explains why — deriving the
id purely from the directory meant every session opened in one repository
shared one plan, "even when those sessions were unrelated." `JCODE_SWARM_ID`
is the explicit opt-in for when you *do* want independently-started root
sessions to share a swarm on purpose.

### Worktree Manager — a term that isn't backed by code

This one is worth calling out plainly, because it's a place where jcode's
own design prose gets ahead of its implementation, and a handbook that
glossed over it would be teaching something false.

`docs/SWARM_ARCHITECTURE.md` describes a "Worktree Manager" role: an agent
that "owns a single worktree scope" and is "responsible for integration
when that worktree scope is done." It reads like a peer of "coordinator."

A full search of the source tree for `WorktreeManager` or `worktree_manager`
finds no struct, no enum variant, no permission check — nothing. The phrase
appears in exactly one place in actual source code: an error message.

```rust
// jcode: crates/jcode-app-core/src/server/comm_sync.rs:346-353
message: "Only the coordinator, worktree manager, or the target session may read full context. Use summary for lightweight access.".to_string(),
```

But the function that message guards checks only two conditions — nothing
resembling a worktree-manager role:

```rust
// jcode: crates/jcode-app-core/src/server/comm_sync.rs:178-192
async fn can_read_full_context(...) -> bool {
    if req_session_id == target_session { return true; }
    let members = swarm_members.read().await;
    members.get(req_session_id).map(|member| member.role == "coordinator").unwrap_or(false)
}
```

Self, or coordinator. That's it. Since `role` is restricted by the tool
schema to exactly `{"agent", "coordinator"}`, there is no third value a
session could even hold to represent "worktree manager." The error text is
aspirational — leftover language from the design intent — not a real branch
in the authorization logic.

Fittingly, jcode's newer design doc (`docs/SWARM_TASK_GRAPH.md`) states the
project's own direction plainly: coordinator and worktree-manager roles are
meant to "demote to scheduler policy, not user-facing concepts" going
forward. So treat "Worktree Manager" as a vocabulary term describing an
*informal* responsibility — an agent can be told, via its spawn prompt, to
own integration for one worktree — not a permission the system enforces.

## The spawn tree, previewed

Every spawned member carries one field, `report_back_to_session_id`,
pointing at whichever session spawned it. That single edge is enough to
reconstruct the entire ancestry chain on demand:

```rust
// jcode: crates/jcode-app-core/src/server/swarm.rs:34-40 (doc comment)
/// The spawner/parent edge is encoded by `report_back_to_session_id`: a child
/// spawned by `P` reports back to `P`. Walking that chain reconstructs the spawn
/// tree without persisting a separate parent field. Cycles (which should never
/// happen) are guarded against with a visited set.
```

The design choice worth noticing here, before Chapter 2 gets into the
mechanics: jcode doesn't maintain a tree data structure that could drift out
of sync with reality. It stores the minimal edge — "who do I report to" —
on each member, and derives the tree by walking it. One field, always
consistent by construction, versus a redundant structure that has to be
kept in sync by hand.

## The architecture, in one picture

```
┌─────────────────────────────────────────────────────────────┐
│  ServerRuntime  — owns every task; structured-concurrency    │
│  shutdown boundary (RuntimeTaskScope: JoinSet + Cancel-      │
│  lationToken)                                                │
│                                                                │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  Shared state (SwarmState)                            │   │
│  │  members · swarms_by_id · plans · coordinators         │   │
│  │  — four independently-locked registries, cheaply       │   │
│  │  Arc-cloned into whichever handler needs them          │   │
│  └──────────────────────────────────────────────────────┘   │
│                                                                │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  One swarm ( = one swarm_id )                          │   │
│  │                                                          │   │
│  │        root (coordinator?)                              │   │
│  │        ├── child A ──┬── grandchild                     │   │
│  │        │              └── grandchild                    │   │
│  │        └── child B                                      │   │
│  │                                                          │   │
│  │  edges = report_back_to_session_id, derived not stored   │   │
│  └──────────────────────────────────────────────────────┘   │
│                                                                │
│  Agent-facing surface: the `swarm` tool — the ONLY way a     │
│  live LLM agent spawns / messages / awaits others            │
│                                                                │
│  Cross-cutting: Bus::global() — process-wide event broadcast │
└─────────────────────────────────────────────────────────────┘
```

The whole thing maps onto TypeScript surprisingly cleanly: one Node
process, N concurrently-running async jobs (or, for one specific spawn mode
covered in Chapter 2, literal `worker_threads`) registered in a parent-side
`Map<id, WorkerHandle>`, with a single shared event emitter for anything
cross-cutting. That's the mental model to carry forward.

## A caveat before you go further

If you go reading `swarm.rs` yourself, you'll run into a function called
`run_swarm_task` built around `try_join_all` — a classic
planner-fans-out-then-awaits-everything shape. It's tempting to assume
that's "what happens when an agent calls the swarm tool to spawn workers."

It isn't. Tracing its callers shows both of them are debug-socket commands
(`swarm_message:`, `swarm_message_async:`), not anything reachable from a
live agent's tool calls. The `swarm` tool's actual `spawn` action goes
through a completely different function, `spawn_swarm_agent`, which is the
real subject of Chapter 2. Keep that distinction in mind — it matters for
every chapter that follows.

---

**Next**: [Chapter 2 — Anatomy of a Spawn](./draft-ch02.md) walks through
exactly what happens, function by function, when a live agent calls
`swarm(action: "spawn")`.
