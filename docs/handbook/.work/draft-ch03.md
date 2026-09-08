# Chapter 3 — Fan-out / Fan-in Patterns

Every "run several things at once, then do something with the results"
problem looks the same from a distance. Up close, jcode actually contains
three genuinely different answers to it, and picking the wrong one produces
code that's either slower than it needs to be or subtly wrong under load.

This chapter walks through all three, back to back, using the exact code
that ships in the repo. The goal isn't "here are three ways to do
concurrency" as a grab-bag — it's to give you a decision procedure. By the
end you should be able to look at a new problem and know which of the three
shapes it wants.

The three questions that separate them:

1. **Is the set of parallel work fixed before anything starts, or discovered
   as things run?**
2. **Do you need the whole batch's result at once, or do you want to react
   as each piece finishes?**
3. **Is "fan-in" a single call that blocks until everything is done, or a
   long-lived loop that wakes up when something happens?**

Keep those three questions in your head — we'll come back to them in the
decision table at the end.

---

## Pattern 1: `try_join_all` planner fan-out

**The plain-English problem:** you want an LLM to break a task into a
handful of independent pieces, run all the pieces at the same time, then
feed all the results back into one final "combine everything" step. This is
the mental picture most people have when they hear "swarm of agents": a
planner splits the work, N workers run concurrently, one final step stitches
the answers together.

One important caveat before we look at the code: this exact function,
`run_swarm_message`, is reachable only from two debug-socket commands
(`swarm_message:` and `swarm_message_async:`) — it is **not** what happens
when a production agent calls the `swarm` tool. The live spawn mechanism
(covered in Chapter 2) is a different code path entirely. We're still using
this function as the chapter's flagship example, because it's real, shipped
code that demonstrates the pattern cleanly — just don't walk away thinking
this is how a normal agent turn fans out work.

With that said, here's the shape. The function does three sequential LLM
calls, with exactly one concurrent step sandwiched in the middle:

1. Ask the LLM to plan: "break this into 2-4 subtasks."
2. Run all the subtasks **at once**.
3. Ask the LLM to integrate all the subtask outputs into one final answer.

Steps 1 and 3 both run on the same locked coordinator agent — they're not
concurrent with anything. Step 2 is the interesting part:

```rust
// jcode: crates/jcode-app-core/src/server/swarm.rs:1657-1671
let task_futures = tasks.iter().map(|task| {
    let agent = agent.clone();
    let working_dir_hint = working_dir_hint.clone();
    let description = task.description.clone();
    let prompt = format!("{working_dir_hint}{}", task.prompt);
    let subagent_type = task
        .subagent_type
        .clone()
        .unwrap_or_else(|| "general".to_string());
    async move {
        let output = run_swarm_task(agent, &description, &subagent_type, &prompt).await?;
        Ok::<(String, String), anyhow::Error>((description, output))
    }
});
let task_outputs = try_join_all(task_futures).await?;
```

Two things happen here. First, `tasks.iter().map(...)` builds an iterator of
**futures** — nothing has actually started running yet, this is just
constructing the work items (async blocks in Rust are lazy; they don't run
until polled). Second, `try_join_all` is what actually launches all of them
concurrently and waits.

Each of those futures, when it eventually runs, calls `run_swarm_task`,
which forks a brand-new session and agent, inherits the parent's exact
model/provider/auth identity so the worker doesn't silently fall back to a
config default, and strips `subagent`/`task`/`todo*` from the child's tool
set — a forked worker in this pattern cannot itself spawn more workers. It
runs exactly one prompt to completion and returns a `Result<String>`.

**Why `try_join_all` specifically:** it needs every future to resolve to the
same `Result<T, E>` shape, and it fails fast — the moment any one subtask
returns an error, the whole call returns that error immediately, and the
other still-in-flight futures are simply dropped (which, on tokio, stops
polling them; nothing explicitly cancels or cleans them up). That's the
correct behavior here specifically because step 3, the integration call, has
no sensible way to combine "2 of the 3 planned subtasks succeeded" into a
coherent answer. The plan assumed 3 outputs; if you only have 2, there's
nothing good to do with them, so failing the whole operation immediately is
the right call.

### The TypeScript equivalent

```typescript
// idiomatic TS equivalent — this code does not exist in jcode
interface SwarmTaskSpec {
  description: string;
  prompt: string;
  subagentType?: string;
}

async function runSwarmMessage(
  coordinator: Agent,
  message: string,
): Promise<string> {
  // Step 1: plan (sequential, runs on the coordinator itself)
  const planText = await coordinator.runOnceCapture(buildPlannerPrompt(message));
  const tasks: SwarmTaskSpec[] = parseSwarmTasks(planText) ?? [
    { description: "Main task", prompt: message },
  ];

  // Step 2: fan-out + fan-in — the concurrent step
  const taskOutputs = await Promise.all(
    tasks.map(async (task) => {
      const worker = await forkWorkerAgent(coordinator, task.subagentType ?? "general");
      const output = await worker.runOnceCapture(task.prompt);
      return [task.description, output] as const;
    }),
  );

  // Step 3: integrate (sequential, back on the coordinator)
  return coordinator.runOnceCapture(buildIntegrationPrompt(taskOutputs));
}
```

`Promise.all` is almost a literal translation of `try_join_all`: same
fail-fast semantics, same "reject as soon as one rejects." The one nuance
worth internalizing: `Promise.all` doesn't actually *cancel* the sibling
promises when one rejects — they keep running in the background, their
results are just ignored. That's exactly the same "fire, don't truly cancel"
behavior as Rust's dropped future. If you assumed `Promise.all` (or
`try_join_all`) stops the other work, both languages will surprise you the
same way.

---

## Pattern 2: `FuturesUnordered` intra-agent batch tool

**The plain-English problem:** a single agent turn wants to fire off several
independent tool calls at once — say, reading five files — as one "batch"
tool invocation, and it wants to show live "3 of 5 done" progress to the UI
as each one finishes, not just block until everything is done.

This is a different shape from Pattern 1 in a way that matters: these
aren't LLM-planned subagents with their own sessions, they're plain tool
calls going through the same tool registry. And the defining requirement —
observing completion *incrementally, in whatever order things actually
finish* — is exactly what `try_join_all`/`Promise.all` cannot give you. Both
of those only resolve once, with everything, after all of it is done.

```rust
// jcode: crates/jcode-app-core/src/tool/batch.rs:282-295
let mut stream: futures::stream::FuturesUnordered<_> = subcalls
    .iter()
    .map(|(i, tool_name, parameters)| {
        let registry = self.registry.clone();
        let i = *i;
        let tool_name = tool_name.clone();
        let parameters = parameters.clone();
        let sub_ctx = ctx.for_subcall(format!("batch-{}-{}", i + 1, tool_name.clone()));
        async move {
            let result = registry.execute(&tool_name, parameters, sub_ctx).await;
            (i, tool_name, result)
        }
    })
    .collect();
```

Each future captures its own original index `i` and hands it back alongside
the result — that's the trick that lets the code recover original order
later, even though the stream itself will yield things in *completion*
order, not submission order.

Then the drain loop:

```rust
// jcode: crates/jcode-app-core/src/tool/batch.rs:300-317
while let Some((i, tool_name, result)) = stream.next().await {
    // ... publishes one BusEvent::BatchProgress per completion
}
```

Each iteration through this loop corresponds to exactly one sub-call
finishing, in whatever order they actually complete in. Each iteration
publishes a `BusEvent::BatchProgress` update — incremented count, name of
the thing that just finished — onto the global event bus (the same bus
covered in Chapter 7; progress from this purely intra-turn concurrency
mechanism is surfaced through the same process-wide channel everything else
in the UI listens to).

Once the loop drains, the code does one more thing worth remembering:

```
results.sort_by_key(|(i, _, _)| *i)   // jcode: tool/batch.rs:319
```

It re-sorts back into original submitted order before formatting the final
text handed back to the LLM. Getting completion-order efficiency for
progress reporting *and* deterministic output order for the final result
isn't a contradiction — it just costs one sort at the end. There's also a
hard cap: `MAX_PARALLEL = 10` (line 10 of the same file) rejects batches
that are too big, and every self-nested `batch` call inside a `batch` call
is rejected outright, so this can't be used to build unbounded fan-out.

### The TypeScript equivalent

There's no single built-in with exactly this shape — `Promise.allSettled`
gets you completion tolerance but still only resolves once, at the end.
The honest idiomatic equivalent is a small hand-rolled drain loop:

```typescript
// idiomatic TS equivalent — this code does not exist in jcode
async function* runBatchIncremental<T>(
  jobs: Array<() => Promise<T>>,
): AsyncGenerator<{ index: number; result: T }> {
  const pending = new Map<number, Promise<{ index: number; result: T }>>();

  jobs.forEach((job, index) => {
    pending.set(
      index,
      job().then((result) => ({ index, result })),
    );
  });

  while (pending.size > 0) {
    const winner = await Promise.race(pending.values());
    pending.delete(winner.index);
    yield winner;
  }
}

// usage: publish progress as each tool call finishes
const results: unknown[] = [];
let completed = 0;
for await (const { index, result } of runBatchIncremental(toolCalls)) {
  completed++;
  bus.publish({ type: "BatchProgress", completed, total: toolCalls.length });
  results[index] = result; // restore original order as we go
}
```

Worth flagging honestly: naive re-racing with `Promise.race` inside a loop
like this is O(n) work per completion, because each call to `race` re-wraps
every remaining promise. It's close enough in spirit to `FuturesUnordered`
to teach the idea, but a production implementation would want a
counter/resolver pattern instead of literally calling `Promise.race` in a
loop. The generator interface is the more important part to internalize:
`for await...of` giving you one result at a time, in completion order, is
the actual shape `FuturesUnordered` provides.

---

## Pattern 3: event-driven fan-in via `broadcast` + `tokio::select!`

**The plain-English problem:** you want to wait until some (or all) of a
set of *already-running* agents reach a target status — but give up after a
deadline, and don't burn CPU polling in a tight loop while you wait.

This is different from both patterns above in a structural way: the
"workers" here weren't started by this function at all. They're independent
swarm members that were spawned earlier and are doing their own work. This
function only *waits* on them — there's no fixed, bounded set of futures to
collect like Pattern 2, and nothing to fan out like Pattern 1.

```rust
// jcode: crates/jcode-app-core/src/server/comm_await.rs:271-300
tokio::select! {
    _ = tokio::time::sleep_until(deadline) => {
        let summary = timeout_summary(&member_statuses);
        finalize_await(&await_members_runtime, &state, false, member_statuses, summary).await;
        return;
    }
    event = event_rx.recv() => {
        match event {
            Ok(event) => {
                if event.swarm_id.as_deref() != Some(swarm_id.as_str()) {
                    continue;
                }
            }
            Err(broadcast::error::RecvError::Lagged(n)) => {
                // Dropped events are recoverable: the loop re-reads
                // member statuses from shared state at the top, so
                // just keep watching instead of orphaning the wait.
                continue;
            }
            Err(broadcast::error::RecvError::Closed) => {
                await_members_runtime.clear_active(&key).await;
                return;
            }
        }
    }
}
```

This lives inside a `loop` (the full function is
`spawn_or_resume_await_members`, `comm_await.rs:209-303`), run as a
detached background task via `tokio::spawn`. Each time around the loop, two
things race: a timeout, and the next incoming `SwarmEvent` off a
`broadcast::Receiver`.

Here's the design point that matters most: **the event itself is never
trusted as the source of truth.** Every iteration starts by re-reading every
watched member's current status directly from shared state
(`awaited_member_statuses`) — not from the event payload — and checking
whether the wait's condition (`"any"` or `"all"` members reached the target
status) is now satisfied. The event's only job is to wake the loop up
sooner than the next timeout tick would.

That design choice is what makes the `Lagged(n)` branch safe to just log and
`continue` past, rather than treat as an error. A `broadcast::Receiver`'s
`recv()` returns `Err(Lagged(n))` when the channel's ring buffer overwrote
`n` messages this particular receiver hadn't read yet — a slow subscriber
missing events under backpressure. Because the loop always re-derives
"are we done" from live shared state rather than from the stream of events,
a missed event costs at most a slightly stale wakeup, never an incorrect
final answer. A comment right at the `Lagged` arm says it plainly: dropped
events are recoverable because the loop re-reads member statuses from
shared state at the top, so it just keeps watching instead of orphaning the
wait.

Contrast the failure model with Pattern 1: `try_join_all` produces a
`Result` and short-circuits on error. This function isn't producing a
`Result` at all — it's a status-matching wait with a deadline. The
`"any"`/`"all"` mode choice is semantically close to `Promise.race` vs.
`Promise.all`, but it's implemented as a polling/event-driven loop against
state owned somewhere else, not a combinator over futures you started
yourself. You reach for this pattern specifically when the things you're
waiting on are not futures you hold — they're independent, already-running
work whose progress lives in shared mutable state.

One more connection worth knowing about, since it bridges into Chapter 7:
when a wait finishes and was started as a background wait with
notification requested, `finalize_await` additionally publishes a
`BusEvent::SwarmAwaitCompleted` onto the *global* event bus — a different,
process-wide channel from the per-swarm `broadcast::Sender<SwarmEvent>`
this function subscribes to. That's the exact hand-off point between the
swarm-scoped fan-in mechanism in this chapter and the global bus covered
next.

### The TypeScript equivalent

```typescript
// idiomatic TS equivalent — this code does not exist in jcode
async function awaitMembers(
  swarmEvents: EventEmitter,
  swarmId: string,
  deadlineMs: number,
  isSatisfied: () => boolean, // re-derives from live shared state
): Promise<{ satisfied: boolean }> {
  return new Promise((resolve) => {
    const timer = setTimeout(() => {
      cleanup();
      resolve({ satisfied: false });
    }, deadlineMs);

    const onEvent = (event: { swarmId: string }) => {
      if (event.swarmId !== swarmId) return; // filter, like the swarm_id check
      // Never trust the event payload directly — always re-check live state.
      if (isSatisfied()) {
        cleanup();
        resolve({ satisfied: true });
      }
      // otherwise: fall through and keep waiting, same as `continue`
    };

    function cleanup() {
      clearTimeout(timer);
      swarmEvents.off("swarm-event", onEvent);
    }

    swarmEvents.on("swarm-event", onEvent);

    // Check once up front in case we're already satisfied.
    if (isSatisfied()) {
      cleanup();
      resolve({ satisfied: true });
    }
  });
}
```

The closest built-in analogy for `broadcast::Receiver` is an `EventEmitter`
sitting on a bounded ring buffer that can drop old events under
backpressure — except Rust's channel tells the receiver explicitly via
`Lagged(n)`, where Node's `EventEmitter` would just silently and invisibly
drop them. The one property worth naming explicitly, because it's the
load-bearing correctness rule of the whole pattern: **always re-derive "are
we done" from source-of-truth state when you wake up — never trust the
event payload as authoritative.** That's what makes a lossy channel safe to
build a correctness-sensitive wait on top of.

---

## When would you reach for each one

| | Pattern 1: `try_join_all` | Pattern 2: `FuturesUnordered` | Pattern 3: `broadcast` + `select!` |
|---|---|---|---|
| What you're waiting on | Futures you just started, whose *set* is fixed up front (a parsed LLM plan) | Futures you just started, whose count is small/bounded (≤10) | Already-running, independently-owned agents you didn't start |
| When you find out about progress | Only once, all at once, at the end | Incrementally, one at a time, in completion order | Incrementally via wake-ups, re-verified against shared state each time |
| Failure behavior | Short-circuits on first error | Every result collected individually — no short-circuit | No `Result` at all — a status-matching wait with a timeout |
| Needs a deadline? | No | No | Yes — the only one of the three built around a timeout |
| Source citation | `swarm.rs:1671` (the call), `1657-1670` (fan-out build) | `tool/batch.rs:282-295` (build), `300-317` (drain) | `comm_await.rs:271-300` (`select!`), `209-303` (whole function) |
| TS analogy | `Promise.all` | Hand-rolled incremental drain (race-in-a-loop or async generator) | `EventEmitter` + timeout race, re-deriving state on every wake |

If the set of parallel work is known up front and you need all of it before
proceeding: **Pattern 1**. If you're firing off a small bounded batch and
want to react as pieces land: **Pattern 2**. If you're waiting on work that's
already running somewhere else, with a deadline, and shouldn't poll: **Pattern
3**.

A quick note on what this chapter didn't cover: the notes this chapter draws
from flagged one honest gap — nobody traced whether an already-spawned
child session's on-disk state gets cleaned up when `try_join_all`
short-circuits and drops its sibling futures. What we know for certain is
the language-level fact (a dropped future stops being polled); whether
jcode does anything further to clean up an abandoned child session is not
something this chapter claims one way or the other.
