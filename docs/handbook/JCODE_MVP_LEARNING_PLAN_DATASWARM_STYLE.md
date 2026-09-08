# Building jcode - MVP Learning Plan (data-swarm notation)
## The same swarm, grown into the codebase you already know

> **This is the plan. Build this one, start to finish.** It goes from the agent
> loop you already have to a working swarm you can point at a real repository.
>
> It uses data-swarm-cli's folder structure, class names, and conventions
> (`query`, `QueryEngine`, `StreamingToolExecutor`, `EventBus`, `buildTool`,
> `sessionContext`, `services/`, `entrypoints/`) rather than inventing new ones,
> so your attention goes to the new material instead of relearning a loop you
> wrote yourself.
>
> Language: JavaScript with JSDoc, matching data-swarm-cli exactly.
> Genuinely new subsystems - the swarm and the task DAG - keep **jcode's own**
> names (`SwarmMember`, `reportBackTo`, `TaskGraph`, `expandNode`,
> `validateGatePass`), because those are the terms the real repository uses and
> you will want to read it afterwards.
>
> **On the companion document.** `JCODE_MVP_LEARNING_PLAN.md` builds the same
> swarm in TypeScript with independent naming. You do not need to build it -
> Parts 1 through 5 are the same logic, and re-typing them to change variable
> names teaches little. It is worth *reading* for two things: its Part 0 shows
> the agent loop built from scratch (which this plan skips, because you have
> one), and the table at the end of this document maps every concept across
> both. Note that Part 6 here has no equivalent there.

---

## WHY THIS VERSION EXISTS

The first plan built a swarm from an empty directory. This one starts from
`data-swarm-cli` and grows a swarm into it - because that codebase already
contains four of the exact seams jcode's swarm needs.

That is not a coincidence, and it is worth understanding before you write a
line:

```
data-swarm-cli already has              which is exactly
--------------------------------------  ------------------------------------------
QueryEngine: "one engine per            One QueryEngine per swarm member.
conversation, messages persist          Spawning = instantiating another one
across turns"                           and recording who its parent is.

sessionContext threaded into every      The carrier for swarm identity. It
tool call, already carrying sessionId   ALREADY has the field the whole spawn
                                        tree is built from.

EventBus: on(type, fn) / emit(...)      The swarm comms bus. DMs, broadcasts,
in-process pub/sub, "keeps the query    and lifecycle events all ride on it
loop, WS server, and UI decoupled"      with barely any change.

StreamingToolExecutor: concurrent-safe  Parallel tool execution, already
tools in parallel, write tools          solved. You will extend it, not
exclusive                               replace it.
```

Your existing architecture is already "an agent is an engine plus its
messages". A swarm is that, N times, with a parent pointer. The rest of this
plan is bookkeeping and one genuinely new idea (the task DAG in Part 4).

**What is new and has no home in data-swarm yet:** cancellation
(`AbortController` appears nowhere in your codebase), member ancestry, the
fan-in strategies, and the DAG. Those are the parts worth slowing down for.

---

## THE ONE DESIGN RULE THAT DECIDES WHETHER THIS WORKS

Before any code. This rule is not stylistic - it is the difference between a
swarm that helps and a swarm that produces confidently broken work.

> **Many agents read. One agent writes.**

The failure mode it prevents is specific: parallel agents make *implicit
decisions* that contradict each other. One worker assumes the config is JSON,
another assumes YAML; each is internally consistent, and the merged result is
incoherent in a way that is very hard to debug because no single agent did
anything wrong.

The industry converged on this the hard way. Cognition (Devin) published
"Don't Build Multi-Agents" arguing exactly this failure, then revisited it ten
months later with a sharper version: multi-agent works *"when writes stay
single-threaded and the additional agents contribute intelligence rather than
actions."* Anthropic's own research system runs **3-5** subagents, not
hundreds - and it is a read-heavy research task. Systems that genuinely run
100+ agents (Kimi's Agent Swarm) use them for **parallel search**, not
collaborative editing.

So the shape you are building toward:

```
                 orchestrator (root)
                 - decides
                 - edits files
                 - owns the plan
                        |
       +----------------+----------------+
       |                |                |
   worker            worker            worker
   read/grep         read/grep         read/grep
   reports back      reports back      reports back
```

Workers investigate and report. The orchestrator decides and writes. Step 40
turns this from a convention into something the code enforces, by handing
different tool sets to different roles.

You *can* let workers write - the machinery supports it, and jcode allows it.
Just do it knowingly, on tasks that touch disjoint files, and expect to spend
your debugging time on merge incoherence rather than on any individual agent.

---

## READ ALONGSIDE: THE HANDBOOK

`SWARM_HANDBOOK.md` explains *why* jcode is built the way it is, with the Rust
walked through chapter by chapter. This plan is the *how*. They are meant to be
read together - one chapter before each part:

| Before you build | Read | What it gives you |
|---|---|---|
| Part 0 | Ch 1 - Why parallel agents | Vocabulary: session, member, coordinator, swarm |
| Part 1 | Ch 2 - Anatomy of a spawn | The real spawn path, ancestry, mode gating, the member cap |
| Part 2 | Ch 3 - Fan-out / fan-in | The three strategies with jcode's actual code, and when each applies |
| Part 2 | Ch 4 - Structured concurrency | Why Rust needs JoinSet + CancellationToken and you need eight lines |
| Part 3 | Ch 5 - Shared state | Why "no locks" and `RwLock` are both true, at different layers |
| Part 3 | Ch 7 - Communication topology | DM vs subtree vs channel, the three read tiers, status-on-reload |
| Part 4 | Ch 6 - The task DAG | The gate machinery, confidence debt, and the worked T0-T7 example |
| Part 6 | Ch 8 - Glossary | The pattern -> file -> primitive cheat sheet |

The handbook's Chapter 6 in particular is worth reading *twice* - once before
Part 4 and once after, because the gate rules only feel inevitable once you
have tried to write them yourself.

---

## SEQUENCE OF IMPLEMENTATION

```
PART 0 - Prepare the loop you already have         5 steps, mostly edits
 1. Fork data-swarm-cli, strip the domain tools
 2. src/services/api/mockClient.js       Deterministic scripted model  <- do not skip
 3. src/tools/BashTool/BashTool.js       spawnSync -> spawn   (this one is load-bearing)
 4. AbortController through sessionContext
 5. StreamingToolExecutor: optional streaming + honest concurrency

PART 1 - One QueryEngine becomes many              jcode Ch2
 6. src/swarm/types.js                   SwarmMember typedefs, lifecycle status
 7. src/swarm/SwarmRegistry.js           The member Map
 8. src/swarm/ancestry.js                reportBackTo edges, depth, subtree
 9. src/swarm/caps.js                    Member cap, worker budget, mode gate
10. src/swarm/Swarm.js                   Owns the QueryEngines
11. src/tools/SpawnTool/SpawnTool.js     The agent-facing spawn
12. src/swarm/lifecycle.js               Reports to parent, reparenting

PART 2 - Running them at once                      jcode Ch3-4
13. src/services/swarm/planFanOut.js     Promise.all
14. src/services/swarm/drainAsCompleted.js  React as each lands
15. src/services/swarm/awaitMembers.js   Event-driven wait with deadline
16. src/swarm/abortTree.js               Parent -> child cancellation
17. src/swarm/interrupt.js               Soft interrupt between turns

PART 3 - Talking to each other                     jcode Ch7
18. src/events/swarmEvents.js            Event type constants + typedefs
19. src/events/EventBus.js               Add a replay buffer
20. src/services/comms/routing.js        DM / subtree broadcast / channel
21. src/services/comms/reads.js          Snapshot / summary / full context
22. src/services/comms/persist.js        Snapshot + rewrite-on-reload

PART 4 - The task DAG                              jcode Ch6
23. src/dag/types.js                     Mode, NodeKind, gateKind, NodeOrigin
24. src/dag/confidence.js                Lenient confidence parsing
25. src/dag/TaskGraph.js                 Private nodes, mutations only
26. src/dag/ops.js                       seed, expandNode
27. src/dag/complete.js                  completeNode + artifact validation
28. src/dag/gates.js                     Root gate, injectFromGate, validateGatePass
29. src/dag/scheduler.js                 readyNodes, dispatch, assembleInput

PART 5 - Run it and watch it                       jcode Ch1, Ch8
30. src/services/swarm/SwarmRunner.js    Drives the DAG with real workers
31. src/components/SwarmView.jsx         Live member tree
32. src/components/DagView.jsx           Graph, seeded vs grown
33. src/screens/REPL.jsx                 Wire the views in
34. src/entrypoints/cli.jsx + demo       End to end, scripted

PART 6 - From demo to daily driver                 the part that makes it usable
35. src/services/permissions/rules.js    Real gate  <- BEFORE any real repo
36. src/services/swarm/planTask.js       Your task -> seed nodes (the front door)
37. src/services/swarm/CostTracker.js    What did that run cost
38. src/services/swarm/Timeline.js       Proof the agents actually overlapped
39. src/tools/ChannelTool/ChannelTool.js Lets agents join channels
40. src/swarm/roles.js                   Enforce single-writer structurally
41. src/entrypoints/cli.jsx              Run it on a real repository (replaces Step 34)
42. src/tools/GraphTool/GraphTool.js     Agents reshape the plan  <- matches jcode
```

---
---

# PART 0 - PREPARE THE LOOP YOU ALREADY HAVE

Five steps, four of which are edits to files you wrote. This is the part the
first plan spent eight steps building from nothing.

---

# STEP 1 - Fork and strip

```bash
cp -r data-swarm-cli jcode-mvp && cd jcode-mvp
rm -rf src/tools/DatabaseSchemaTool src/tools/QueryExecutorTool src/tools/GA4Tools
rm -rf src/db
```

Those are data-swarm's domain: schema inspection, SQL execution, GA4. None of
it is about agents. Keep `BashTool`, `WriteFileTool`, and `AskUserQuestionTool`
- the last one is worth keeping specifically because a swarm makes HITL more
interesting, not less.

Swarm workers need to read and search, so add two more in your existing shape -
one folder per tool, `buildTool({...})`:

```javascript
// src/tools/ReadFileTool/ReadFileTool.js
import { z } from 'zod'
import { buildTool } from '../../Tool.js'
import fs from 'node:fs/promises'
import path from 'node:path'

export const ReadFileTool = buildTool({
  name: 'read_file',
  description: 'Read a file from the session folder.',
  inputSchema: z.object({
    path: z.string().describe('Path relative to the session folder'),
  }),

  async call(input, context) {
    const root = context?.sessionContext?.folderPath ?? process.cwd()
    const full = path.resolve(root, input.path)
    const content = await fs.readFile(full, 'utf8')
    return { data: content }
  },

  async checkPermissions() { return { granted: true } },

  // Reads never conflict, so they may overlap freely.
  isConcurrencySafe: () => true,
  isReadOnly: () => true,
})
```

```javascript
// src/tools/GrepTool/GrepTool.js
import { z } from 'zod'
import { buildTool } from '../../Tool.js'
import { spawn } from 'node:child_process'

export const GrepTool = buildTool({
  name: 'grep',
  description: 'Search the session folder for a pattern.',
  inputSchema: z.object({
    pattern: z.string(),
    path: z.string().optional(),
  }),

  async call(input, context) {
    const cwd = context?.sessionContext?.folderPath ?? process.cwd()
    const signal = context?.sessionContext?.signal

    const output = await new Promise((resolve) => {
      const child = spawn('grep', ['-rn', input.pattern, input.path ?? '.'], { cwd })
      let buf = ''
      child.stdout.on('data', (c) => { buf += c })
      signal?.addEventListener('abort', () => child.kill(), { once: true })
      // grep exits 1 on "no matches", which is not an error here.
      child.on('close', () => resolve(buf))
      child.on('error', () => resolve(''))
    })

    const lines = output.split('\n').filter(Boolean).slice(0, 50)
    return { data: lines.join('\n') || '(no matches)' }
  },

  async checkPermissions() { return { granted: true } },

  isConcurrencySafe: () => true,
  isReadOnly: () => true,
})
```

**Both are `isReadOnly: () => true`, and that is load-bearing.** Your
`StreamingToolExecutor` reads that field to decide what may run in parallel. Get
it wrong and a worker reading four files does so one at a time, behind the write
gate.

**And one more tool, which is the difference between a demo and something you
would actually use.** `WriteFileTool` replaces a whole file. On a 2,000-line
source file that means the model reproduces all 2,000 lines to change three of
them - slow, expensive, and it *will* silently drop code on the way through.
Real coding agents edit by replacing an exact substring:

```javascript
// src/tools/EditFileTool/EditFileTool.js
import { z } from 'zod'
import { buildTool } from '../../Tool.js'
import fs from 'node:fs/promises'
import path from 'node:path'

export const EditFileTool = buildTool({
  name: 'edit_file',
  description:
    'Replace an exact string in a file. old_string must appear EXACTLY once - ' +
    'include surrounding lines to make it unique.',

  inputSchema: z.object({
    path: z.string(),
    old_string: z.string().describe('Exact text to replace, including whitespace'),
    new_string: z.string().describe('Replacement text'),
  }),

  async call(input, context) {
    const root = context?.sessionContext?.folderPath ?? process.cwd()
    const full = path.resolve(root, input.path)
    const before = await fs.readFile(full, 'utf8')

    const occurrences = before.split(input.old_string).length - 1

    // Refusing ambiguity is the whole value. A model that edits the wrong one
    // of three matches produces a bug that looks like it came from nowhere.
    if (occurrences === 0) {
      return {
        data: `No match in ${input.path}. The file may have changed since you read it - ` +
              `read it again and copy the exact text including indentation.`,
      }
    }
    if (occurrences > 1) {
      return {
        data: `old_string appears ${occurrences} times in ${input.path}. ` +
              `Include more surrounding lines so it matches exactly once.`,
      }
    }

    await fs.writeFile(full, before.replace(input.old_string, input.new_string), 'utf8')
    return { data: `Edited ${input.path}` }
  },

  async checkPermissions() { return { granted: true } },

  isConcurrencySafe: () => false,
  isReadOnly: () => false,
})
```

Add `EditFileTool` to `allTools` under the File group. Note that it returns a
*message* rather than throwing when the match is ambiguous - the model reads
that string and retries with more context, which is the same
error-strings-are-control-flow idea you will meet again in Step 28's gates.

Then trim `src/tools.js` to match:

```javascript
// src/tools.js
import { ReadFileTool }        from './tools/ReadFileTool/ReadFileTool.js'
import { WriteFileTool }       from './tools/WriteFileTool/WriteFileTool.js'
import { GrepTool }            from './tools/GrepTool/GrepTool.js'
import { BashTool }            from './tools/BashTool/BashTool.js'
import { AskUserQuestionTool } from './tools/AskUserQuestionTool/AskUserQuestionTool.js'

/**
 * All tools registered and available to the query loop.
 *
 * Tool groups:
 *   File    - read, write, search
 *   Shell   - run commands
 *   HITL    - clarifying questions to the user
 *   Swarm   - spawn/message teammates (added in Part 1)
 */
export const allTools = [
  ReadFileTool,
  WriteFileTool,
  GrepTool,
  BashTool,
  AskUserQuestionTool,
]
```

Everything else - `query.js`, `QueryEngine.js`, `Tool.js`,
`StreamingToolExecutor.js`, `EventBus.js`, `state/store.js`, `screens/REPL.jsx`,
`entrypoints/cli.jsx` - stays exactly where it is.

---

# STEP 2 - `src/services/api/mockClient.js`

**Why this is Step 2 and not Step 30:** you are about to run ten agents at
once. If every run costs money, takes ninety seconds, and makes slightly
different choices each time, you will not iterate - you will guess. A scripted
model turns a swarm run into a unit test: instant, free, identical every time.

**The contract to match** is `createApiStream` in `src/services/api/client.js`,
which yields exactly two event shapes:

```
{ type: 'assistant_message', message }
{ type: 'tool_use_block', block, assistantMessage }
```

Match it precisely and `query.js` cannot tell the difference.

```javascript
// src/services/api/mockClient.js

/**
 * @typedef {Object} ScriptedTurn
 * @property {string} [text]        Assistant text for this turn
 * @property {Array<{name: string, input: Object}>} [toolCalls]
 */

/**
 * @callback Script
 * @param {number} turnIndex   0 on the first turn of this engine, then 1, 2...
 * @param {Object} params      Same params createApiStream receives
 * @returns {ScriptedTurn}
 */

let toolCounter = 0
const nextToolId = () => `mock_tool_${++toolCounter}`

/**
 * Builds a drop-in replacement for createApiStream.
 *
 * Each call to the returned function is one turn. The counter is per-factory,
 * so give every QueryEngine its own mock if you want per-agent turn indexes -
 * which is what createMockClientFactory below does for you.
 *
 * @param {Script} script
 * @param {{ latencyMs?: number }} [options]
 */
export function createMockClient(script, options = {}) {
  const latencyMs = options.latencyMs ?? 0
  let turnIndex = 0

  return async function* mockApiStream(params) {
    const turn = script(turnIndex++, params) ?? {}

    if (latencyMs > 0) {
      await new Promise(r => setTimeout(r, latencyMs))
    }

    const toolCalls = turn.toolCalls ?? []

    const assistantMessage = {
      type: 'assistant',
      content: [
        ...(turn.text ? [{ type: 'text', text: turn.text }] : []),
        ...toolCalls.map(c => ({
          type: 'tool_use',
          id: nextToolId(),
          name: c.name,
          input: c.input,
        })),
      ],
      stopReason: toolCalls.length > 0 ? 'tool_use' : 'end_turn',
      usage: {
        inputTokens: 100,
        outputTokens: 50,
        cacheReadInputTokens: 0,
      },
    }

    yield { type: 'assistant_message', message: assistantMessage }

    for (const block of assistantMessage.content) {
      if (block.type === 'tool_use') {
        yield { type: 'tool_use_block', block, assistantMessage }
      }
    }
  }
}

/**
 * One script, many agents, each with its own turn counter.
 * Swarm members each need their own instance or they share turn state.
 */
export function createMockClientFactory(script, options = {}) {
  return () => createMockClient(script, options)
}
```

**Give the mock 5-30ms of latency whenever you are testing the swarm.** At zero
latency everything settles inside one microtask tick, you never observe
interleaving, your parallel agents look serial, and you go hunting a bug that
does not exist.

**Now make `query.js` accept an injected client.** One line at the top, one
line at the call site:

```javascript
// src/query.js - add to the destructured params
const {
  tools,
  getAppState,
  setAppState,
  systemPrompt,
  maxTurns = 50,
  onMessagesUpdate,
  sessionContext = {},
  createStream = createApiStream,   // <- add this
} = params

// ...and at the call site, replace createApiStream(...) with:
const stream = createStream({ messages: compacted.messages, tools, systemPrompt })
```

Default-to-real, override-for-tests. `QueryEngine` passes it through from its
config in Part 1.

---

# STEP 3 - Fix `BashTool`: `spawnSync` to `spawn`

**This step is load-bearing, and it is easy to skip because the tool already
works.**

Your current `BashTool` calls `spawnSync` (`src/tools/BashTool/BashTool.js`).
In a single-agent CLI that is fine - nothing else wants the thread. In a swarm
it is fatal: `spawnSync` blocks Node's event loop, so while one worker runs
`npm test`, **every other agent is frozen**. Not slowed. Stopped. Your swarm
will look parallel in the code and behave serial at runtime, and the symptom
(everything is mysteriously sequential) points nowhere near the cause.

```javascript
// src/tools/BashTool/BashTool.js
import { z } from 'zod'
import { buildTool } from '../../Tool.js'
import { spawn } from 'child_process'

export const BashTool = buildTool({
  name: 'bash',
  description: 'Execute a shell command in the local environment.',
  inputSchema: z.object({
    command: z.string().describe('Shell command to execute'),
    cwd: z.string().optional().describe('Working directory'),
    timeout: z.number().optional().default(30_000).describe('Timeout in milliseconds'),
  }),

  async call(input, context) {
    const cwd = input.cwd || context?.sessionContext?.folderPath
    if (!cwd) {
      throw new Error('No working directory: session has no folderPath. Bootstrap a session first.')
    }

    const env = { ...process.env }
    const venvPath = context?.sessionContext?.venvPath
    if (venvPath) {
      env.VIRTUAL_ENV = venvPath
      env.PATH = `${venvPath}/bin:${env.PATH || ''}`
    }

    // Non-blocking. One agent's slow command no longer freezes the swarm.
    const signal = context?.sessionContext?.signal

    return await new Promise((resolve, reject) => {
      const child = spawn('bash', ['-c', input.command], { cwd, env })

      let stdout = ''
      let stderr = ''
      child.stdout.on('data', (c) => { stdout += c })
      child.stderr.on('data', (c) => { stderr += c })

      const onAbort = () => child.kill('SIGINT')
      signal?.addEventListener('abort', onAbort, { once: true })

      const timer = setTimeout(() => child.kill('SIGTERM'), input.timeout ?? 30_000)

      child.on('error', (err) => {
        clearTimeout(timer)
        signal?.removeEventListener('abort', onAbort)
        reject(err)
      })

      child.on('close', (code) => {
        clearTimeout(timer)
        signal?.removeEventListener('abort', onAbort)
        resolve({
          data: { stdout, stderr, exitCode: code ?? 0 },
        })
      })
    })
  },

  async checkPermissions(_input, _context) {
    return { granted: true }
  },

  isConcurrencySafe: () => false,
  isReadOnly: () => false,
})
```

Note the return shape is unchanged: `{ data: { stdout, stderr, exitCode } }`.
Nothing downstream notices.

**While you are here, a quirk in your own code worth knowing.** Your tools
declare both `isConcurrencySafe` and `isReadOnly`, but `StreamingToolExecutor`
only ever reads `isReadOnly`:

```javascript
// src/services/tools/StreamingToolExecutor.js, in addTool()
const isConcurrencySafe = definition
  ? definition.isReadOnly(block.input)     // <- isConcurrencySafe is never called
  : false
```

`isConcurrencySafe` is currently dead code. Keep both fields (jcode makes the
same distinction and it is a real one - a read can be safe to *parallelise*
while still not being *read-only* in the caching sense), but know that today
only `isReadOnly` decides scheduling.

---

# STEP 4 - AbortController through `sessionContext`

**What is missing:** `AbortController` appears nowhere in data-swarm-cli. For
one interactive agent you can live without it. For a swarm you cannot - killing
a subtree, enforcing a deadline, and shutting down cleanly all need a signal
that flows parent to child.

**Where it goes:** `sessionContext`, which already reaches every tool. Zero new
plumbing, which is exactly why this codebase was a good starting point.

**What jcode does:** a `CancellationToken` per task scope
(`crates/jcode-app-core/src/server/runtime.rs:27-79`) plus an `InterruptSignal`
built from an `AtomicBool` and a `Notify`, with a careful guard against a
lost-wakeup race (`crates/jcode-agent-runtime/src/lib.rs:32-40`, `:92-106`).
All of that machinery exists because Rust is multi-threaded. `AbortController`
gives you the same guarantees for free.

```javascript
// src/services/session/createSessionContext.js
let sessionSeq = 0

/**
 * @typedef {Object} SessionContext
 * @property {string} sessionId
 * @property {string} folderPath
 * @property {string} [userId]
 * @property {string} [venvPath]
 * @property {AbortSignal} signal
 * @property {AbortController} abort
 * @property {import('../../swarm/Swarm.js').Swarm} [swarm]   Added in Part 1
 * @property {import('../../events/EventBus.js').EventBus} [bus]
 */

/**
 * Creates a session context. If a parent signal is supplied, cancellation
 * flows downhill: aborting the parent aborts this session and, transitively,
 * everything it spawned.
 *
 * @returns {SessionContext}
 */
export function createSessionContext(options = {}) {
  const abort = new AbortController()

  if (options.parentSignal) {
    if (options.parentSignal.aborted) {
      abort.abort()
    } else {
      options.parentSignal.addEventListener('abort', () => abort.abort(), { once: true })
    }
  }

  return {
    sessionId: options.sessionId ?? `s${++sessionSeq}_${Math.random().toString(36).slice(2, 7)}`,
    folderPath: options.folderPath ?? process.cwd(),
    userId: options.userId,
    venvPath: options.venvPath,
    abort,
    signal: abort.signal,
    swarm: options.swarm,
    bus: options.bus,
  }
}
```

**Then teach the loop to notice.** In `src/query.js`, at the top of the
`while (turns < maxTurns)` body:

```javascript
  while (turns < maxTurns) {
    if (sessionContext.signal?.aborted) {
      console.log('\n[aborted] Session cancelled, stopping loop.')
      break
    }
    turns++
```

That is the whole cancellation story for the loop. The tools handle their own
mid-flight abort, which is why Step 3 wired `signal` into `BashTool`.

---

# STEP 5 - `StreamingToolExecutor`: optional streaming, and honest concurrency

Two changes to a file you already have.

**Change one: let a tool stream progress.** A swarm worker that runs for forty
seconds should be able to say what it is doing. Your `call(input, context)`
contract stays exactly as it is - existing tools are untouched - but a tool may
now be written as an async generator that *yields* progress strings and
*returns* the usual `{ data }`.

The detection is a one-liner: calling an async generator function returns an
async generator synchronously, while a plain async function returns a Promise.

**Change two: understand what your executor actually does,** because it is
subtler than it looks and you are about to depend on it.

```javascript
async processQueue() {
  for (const tool of this.tools) {
    if (tool.status !== 'queued') continue
    if (!this.canExecute(tool.isConcurrencySafe)) {
      if (!tool.isConcurrencySafe) break
      continue
    }
    await this.executeTool(tool)      // <- awaits, inside the loop
  }
}
```

A single pass through `processQueue` runs tools **one at a time**. Parallelism
does not come from that loop - it comes from `addTool` firing
`void this.processQueue()` again for each new block that arrives mid-stream, so
several `processQueue` calls overlap and each picks up a different queued tool.
It works, and it is genuinely clever, but the concurrency is *emergent* rather
than stated, which makes it hard to reason about when six agents are running.

Keep the behaviour, make it explicit:

```javascript
// src/services/tools/StreamingToolExecutor.js
import { findToolByName } from '../../Tool.js'

/**
 * Executes tools as they stream in, with concurrency control.
 *
 * - Concurrent-safe tools (isReadOnly) run in parallel
 * - Write tools run exclusively - queue blocks
 * - Results are emitted in declaration order (not completion order)
 *
 * Concurrency note: processQueue() starts every eligible tool without awaiting
 * it, and records the promise. Overlapping calls are therefore safe, and the
 * parallelism is visible in the code rather than emerging from call timing.
 */
export class StreamingToolExecutor {
  constructor(toolDefinitions, getAppState, setAppState, sessionContext = {}) {
    this.toolDefinitions = toolDefinitions
    this.getAppState = getAppState
    this.setAppState = setAppState
    this.sessionContext = sessionContext
    this.tools = []
    this.onProgress = sessionContext.onToolProgress ?? null
  }

  addTool(block, assistantMessage) {
    const definition = findToolByName(this.toolDefinitions, block.name)
    const isConcurrencySafe = definition ? definition.isReadOnly(block.input) : false

    this.tools.push({
      id: block.id,
      block,
      assistantMessage,
      status: 'queued',
      isConcurrencySafe,
      promise: null,
      results: null,
    })

    this.processQueue()
  }

  canExecute(isConcurrencySafe) {
    const executing = this.tools.filter(t => t.status === 'executing')
    return (
      executing.length === 0 ||
      (isConcurrencySafe && executing.every(t => t.isConcurrencySafe))
    )
  }

  /** Starts everything eligible. Synchronous on purpose - it must not await. */
  processQueue() {
    for (const tool of this.tools) {
      if (tool.status !== 'queued') continue
      if (!this.canExecute(tool.isConcurrencySafe)) {
        if (!tool.isConcurrencySafe) break
        continue
      }
      this.startTool(tool)
    }
  }

  startTool(tool) {
    tool.status = 'executing'
    const definition = findToolByName(this.toolDefinitions, tool.block.name)

    tool.promise = (async () => {
      if (!definition) {
        tool.results = [errorResult(tool.block.id, `No tool: ${tool.block.name}`)]
        return
      }

      const permResult = await definition.checkPermissions(tool.block.input, {
        getAppState: this.getAppState,
        setAppState: this.setAppState,
      })

      if (!permResult.granted) {
        tool.results = [errorResult(tool.block.id, `Permission denied: ${permResult.reason}`)]
        return
      }

      try {
        const result = await this.invoke(definition, tool)
        tool.results = [toolResultMessage(tool.block.id, result.data)]
      } catch (err) {
        console.error(`[Tool Error]: ${tool.block.name} - ${err.message}`)
        tool.results = [errorResult(tool.block.id, String(err))]
      }
    })().finally(() => {
      tool.status = 'completed'
      this.processQueue()
    })
  }

  /**
   * Calls a tool. Supports both shapes:
   *   plain async  -> returns { data }
   *   async gen    -> yields progress strings, returns { data }
   */
  async invoke(definition, tool) {
    const context = {
      getAppState: this.getAppState,
      setAppState: this.setAppState,
      sessionContext: this.sessionContext,
    }

    const out = definition.call(tool.block.input, context)

    if (out && typeof out[Symbol.asyncIterator] === 'function') {
      while (true) {
        const step = await out.next()
        if (step.done) return step.value ?? { data: '' }
        this.onProgress?.(tool.block.id, tool.block.name, step.value)
      }
    }

    return await out
  }

  async *getRemainingResults() {
    for (const tool of this.tools) {
      if (tool.promise) await tool.promise
      for (const msg of tool.results ?? []) {
        yield msg
      }
    }
  }
}

function toolResultMessage(toolUseId, content) {
  return {
    type: 'tool_result',
    toolUseId,
    content: typeof content === 'string' ? content : JSON.stringify(content),
  }
}

function errorResult(toolUseId, error) {
  return {
    type: 'tool_result',
    toolUseId,
    content: error,
    isError: true,
  }
}
```

**What changed and why it matters:** `startTool` no longer awaits inside
`processQueue`, so eligible tools genuinely start together and the write-queue
gate still holds. `getRemainingResults` is untouched, so results still come
back in declaration order - which is what the Anthropic API requires when it
matches `tool_result` blocks to `tool_use` ids.

**Checkpoint - Part 0 works.** Run your existing REPL against the mock:

```javascript
// scratch.mjs
import { QueryEngine } from './src/QueryEngine.js'
import { createMockClient } from './src/services/api/mockClient.js'
import { createSessionContext } from './src/services/session/createSessionContext.js'
import { allTools } from './src/tools.js'
import { createStore } from './src/state/store.js'
import { getDefaultAppState } from './src/state/AppStateStore.js'

const store = createStore(getDefaultAppState())

const engine = new QueryEngine({
  tools: allTools,
  getAppState: store.getState,
  setAppState: store.setState,
  systemPrompt: 'You are helpful.',
  sessionContext: createSessionContext({ folderPath: process.cwd() }),
  createStream: createMockClient((turn) =>
    turn === 0
      ? { text: 'Let me look.', toolCalls: [{ name: 'bash', input: { command: 'echo hello' } }] }
      : { text: 'The command printed hello.' }
  ),
})

for await (const event of engine.submitMessage('Say hello via bash')) {
  console.log(JSON.stringify(event).slice(0, 200))
}
```

For that to work, `QueryEngine` must forward two new config fields. Two lines
in `src/QueryEngine.js`:

```javascript
    yield* query({
      messages: this.messages,
      tools: this.config.tools,
      getAppState: this.config.getAppState,
      setAppState: this.config.setAppState,
      systemPrompt: this.config.systemPrompt,
      maxTurns: this.config.maxTurns,
      sessionContext: this.sessionContext,
      createStream: this.config.createStream,     // <- add
      onMessagesUpdate: (msgs) => {
        this.messages = msgs
      },
    })
```

You should see an assistant message, a `tool_result` containing `hello`, and
the loop stopping. **Do not continue until this runs** - everything after this
assumes the loop, the mock, and the session context all work together.

---
---

# PART 1 - ONE QueryEngine BECOMES MANY

The whole trick, stated once: **a swarm member is a `QueryEngine` plus a sticky
note saying who to report back to.**

There is no Agent class, no actor framework, no scheduler thread. jcode
reconstructs its entire spawn tree by following one string field
(`report_back_to_session_id`). Your `sessionContext` already carries
`sessionId`; you are about to add its parent.

---

# STEP 6 - `src/swarm/types.js`

**What jcode does:** members live in a registry keyed by session id, each
carrying status, role, and `report_back_to_session_id`
(`crates/jcode-app-core/src/server/comm_session.rs:502-519` inserts exactly
these fields).

**Why the tree is derived, never stored:** ownership ("may I stop this?"),
broadcast reach, and report routing are all computed from parent pointers. One
field cannot desynchronise from itself. A second structure holding the same
tree can, and eventually will.

```javascript
// src/swarm/types.js

/**
 * Mirrors jcode's lifecycle states (docs/SWARM_ARCHITECTURE.md,
 * "Agent Lifecycle States").
 *
 * @typedef {'spawned'|'ready'|'running'|'blocked'|'completed'|'failed'|'stopped'|'crashed'} MemberStatus
 */

/** @typedef {'coordinator'|'agent'} MemberRole */

/**
 * Spawn modes, enforced in Step 9.
 *   adhoc / light - only the root may spawn (one level of fan-out)
 *   deep          - any member may spawn, recursively
 * jcode: SwarmSpawnMode, crates/jcode-config-types/src/lib.rs:643-657
 *
 * @typedef {'adhoc'|'light'|'deep'} SpawnMode
 */

/**
 * @typedef {Object} SwarmMember
 * @property {string} sessionId
 * @property {string} swarmId
 * @property {string|null} reportBackTo  The whole tree derives from this. null = root.
 * @property {MemberRole} role
 * @property {MemberStatus} status
 * @property {string} friendlyName
 * @property {string} [taskLabel]
 * @property {string} [latestReport]
 * @property {number} createdAt
 */

/** @type {MemberStatus[]} */
export const TERMINAL_STATUSES = ['completed', 'failed', 'stopped', 'crashed']

/** @param {MemberStatus} status */
export function isTerminalStatus(status) {
  return TERMINAL_STATUSES.includes(status)
}
```

---

# STEP 7 - `src/swarm/SwarmRegistry.js`

**What jcode does:**

```rust
swarm_members: Arc<RwLock<HashMap<String, SwarmMember>>>
```

every read `.read().await`, every write `.write().await`.

**What you write:** a `Map`.

**Sit with this one, because it is the clearest Rust-tax moment in the build.**
jcode needs that lock because tokio may run tasks on several OS threads, so two
agents genuinely can touch the map in the same instant. Node runs your
JavaScript on one thread: between any two lines of synchronous code nothing
else executes, so a `Map.set` cannot interleave with a `Map.get`. The lock
defends against a hazard your runtime does not have.

What Node does **not** protect you from is the `await` boundary. State can
change across any `await`, so "read, await something, then write based on the
old read" is still a bug. That is a logic error, not a data race - and Step 15
is where it bites.

```javascript
// src/swarm/SwarmRegistry.js

/**
 * The swarm's member table plus the QueryEngine that backs each member.
 * jcode holds the equivalent behind an RwLock; a plain Map is enough here.
 */
export class SwarmRegistry {
  constructor() {
    /** @type {Map<string, import('./types.js').SwarmMember>} */
    this.members = new Map()
    /** @type {Map<string, import('../QueryEngine.js').QueryEngine>} */
    this.engines = new Map()
    /** @type {Map<string, import('../services/session/createSessionContext.js').SessionContext>} */
    this.contexts = new Map()
  }

  add(member, engine, sessionContext) {
    this.members.set(member.sessionId, member)
    this.engines.set(member.sessionId, engine)
    this.contexts.set(member.sessionId, sessionContext)
  }

  get(sessionId) {
    return this.members.get(sessionId)
  }

  engine(sessionId) {
    return this.engines.get(sessionId)
  }

  context(sessionId) {
    return this.contexts.get(sessionId)
  }

  all() {
    return [...this.members.values()]
  }

  update(sessionId, patch) {
    const existing = this.members.get(sessionId)
    if (!existing) return
    this.members.set(sessionId, { ...existing, ...patch })
  }

  setStatus(sessionId, status) {
    this.update(sessionId, { status })
  }

  remove(sessionId) {
    const member = this.members.get(sessionId)
    this.members.delete(sessionId)
    this.engines.delete(sessionId)
    this.contexts.delete(sessionId)
    return member
  }

  count() {
    return this.members.size
  }
}
```

---

# STEP 8 - `src/swarm/ancestry.js`

**What jcode does:** walks `report_back_to_session_id` to rebuild the tree on
demand. Ownership is defined as *is this in the subtree I spawned*
(`crates/jcode-app-core/src/server/swarm.rs:995-1213`).

```javascript
// src/swarm/ancestry.js

export function parentOf(registry, sessionId) {
  const member = registry.get(sessionId)
  return member?.reportBackTo ? registry.get(member.reportBackTo) : undefined
}

export function childrenOf(registry, sessionId) {
  return registry.all().filter(m => m.reportBackTo === sessionId)
}

/** Root is depth 0. Guards against cycles rather than hanging. */
export function depthOf(registry, sessionId) {
  let depth = 0
  let cursor = registry.get(sessionId)
  const seen = new Set()

  while (cursor?.reportBackTo) {
    if (seen.has(cursor.sessionId)) break
    seen.add(cursor.sessionId)
    cursor = registry.get(cursor.reportBackTo)
    depth++
  }
  return depth
}

/** Every transitive descendant, excluding sessionId itself. Breadth-first. */
export function subtreeOf(registry, sessionId) {
  const out = []
  const queue = [sessionId]
  const seen = new Set([sessionId])

  while (queue.length > 0) {
    const current = queue.shift()
    for (const child of childrenOf(registry, current)) {
      if (seen.has(child.sessionId)) continue
      seen.add(child.sessionId)
      out.push(child)
      queue.push(child.sessionId)
    }
  }
  return out
}

/** "Do I own this agent?" - the authorization primitive. */
export function isInSubtree(registry, ancestorId, candidateId) {
  return subtreeOf(registry, ancestorId).some(m => m.sessionId === candidateId)
}

export function rootsOf(registry) {
  return registry.all().filter(m => m.reportBackTo === null)
}
```

**Why the cycle guards:** `reportBackTo` gets rewritten during reparenting
(Step 12). A bug there turns `depthOf` into an infinite loop that hangs the
process with no error and no stack. Four lines convert a hang into a
wrong-but-visible number.

---

# STEP 9 - `src/swarm/caps.js`

**What jcode does:** two independent limits plus a mode gate. An absolute cap
(`MAX_SWARM_MEMBERS = 1000`), a configurable live-worker budget, and the rule
that only deep-mode roots may spawn recursively - in normal and light swarms a
worker cannot spawn at all (docs/SWARM_ARCHITECTURE.md, "Mode-gated spawning").

**Why the mode gate exists:** recursion is exponential. A worker that spawns
three workers that each spawn three is forty agents from one prompt, each
costing tokens. Making recursion opt-in bounds casual use *by construction*
rather than by hoping the model is sensible.

```javascript
// src/swarm/caps.js
import { isTerminalStatus } from './types.js'
import { depthOf } from './ancestry.js'

/** jcode's absolute ceiling is 1000. Yours is small so mistakes stay cheap. */
export const MAX_SWARM_MEMBERS = 50

/**
 * @typedef {Object} SpawnPolicy
 * @property {import('./types.js').SpawnMode} mode
 * @property {number} maxLiveWorkers   How many members may be non-terminal at once
 */

export function liveWorkerCount(registry) {
  return registry.all().filter(m => !isTerminalStatus(m.status)).length
}

/**
 * @returns {{ ok: true } | { ok: false, reason: string }}
 */
export function canSpawn(registry, requesterId, policy) {
  const member = registry.get(requesterId)
  if (!member) return { ok: false, reason: 'Requester is not a swarm member' }

  if (registry.count() >= MAX_SWARM_MEMBERS) {
    return { ok: false, reason: `Swarm is at its cap of ${MAX_SWARM_MEMBERS} members` }
  }

  const live = liveWorkerCount(registry)
  if (live >= policy.maxLiveWorkers) {
    return {
      ok: false,
      reason: `Live-worker budget exhausted (${live}/${policy.maxLiveWorkers}); wait for one to finish`,
    }
  }

  if (policy.mode !== 'deep' && depthOf(registry, requesterId) > 0) {
    return {
      ok: false,
      reason: `Only the root may spawn in '${policy.mode}' mode. Do the work yourself and report back.`,
    }
  }

  return { ok: true }
}
```

**Write refusals for the model to read, not for a log file.** "Denied" makes
the model retry the same call until it burns `maxTurns`. "Do the work yourself
and report back" redirects it. Error strings aimed at an LLM are control flow.

---

# STEP 10 - `src/swarm/Swarm.js`

The centre of Part 1: the object that owns many `QueryEngine`s.

**What jcode does:** `spawn_swarm_agent`
(`crates/jcode-app-core/src/server/comm_session.rs:557-827`) creates a session,
inserts a member, then **detaches the child's first turn** with `tokio::spawn` -
the parent's tool call returns immediately.

**Two things to notice as you write it:**

1. **Keep the detached promise.** The moment you write `void run()` you have
   created work nobody can await, cancel, or observe. jcode has a whole type
   for this problem - `RuntimeTaskScope`
   (`crates/jcode-app-core/src/server/runtime.rs:27-79`), whose doc comment
   says outright that dropping a task handle detaches it, so accepted work must
   never discard the handle. Your `Map` of promises is that idea in one line.
2. **The registry authorizes, not the caller.** `canSpawn` is evaluated against
   the requester's position in the tree, which the requester cannot lie about
   because it comes from the registry rather than from tool input.

```javascript
// src/swarm/Swarm.js
import { QueryEngine } from '../QueryEngine.js'
import { EventBus } from '../events/EventBus.js'
import { SwarmRegistry } from './SwarmRegistry.js'
import { createSessionContext } from '../services/session/createSessionContext.js'
import { canSpawn } from './caps.js'
import { reportToParent } from './lifecycle.js'

let nameCounter = 0

/**
 * @typedef {Object} SwarmOptions
 * @property {string} swarmId
 * @property {string} folderPath
 * @property {Array} tools
 * @property {() => any} getAppState
 * @property {(fn: Function) => void} setAppState
 * @property {(member: import('./types.js').SwarmMember) => string} systemPromptFor
 * @property {import('./caps.js').SpawnPolicy} policy
 * @property {() => Function} createStreamFactory   Returns a fresh createStream per member
 * @property {EventBus} [bus]
 */

export class Swarm {
  /** @param {SwarmOptions} options */
  constructor(options) {
    this.options = options
    this.registry = new SwarmRegistry()
    this.bus = options.bus ?? new EventBus()
    /** Detached turns, kept addressable. The JS answer to RuntimeTaskScope. */
    this.running = new Map()
  }

  /** Creates the depth-0 member everything else descends from. */
  createRoot(prompt) {
    const member = this.#createMember({
      reportBackTo: null,
      role: 'coordinator',
      friendlyName: 'root',
      status: 'ready',
      parentSignal: undefined,
    })
    this.rootPrompt = prompt
    return member
  }

  /**
   * @param {{ requesterId: string, prompt: string, taskLabel?: string, friendlyName?: string }} request
   * @returns {{ ok: true, member: import('./types.js').SwarmMember } | { ok: false, reason: string }}
   */
  spawn(request) {
    const decision = canSpawn(this.registry, request.requesterId, this.options.policy)
    if (!decision.ok) return decision

    const parentContext = this.registry.context(request.requesterId)

    const member = this.#createMember({
      reportBackTo: request.requesterId,
      role: 'agent',
      friendlyName: request.friendlyName ?? `worker-${++nameCounter}`,
      taskLabel: request.taskLabel,
      status: 'spawned',
      parentSignal: parentContext?.signal,
    })

    // Fire and forget - but keep the handle.
    this.running.set(member.sessionId, this.runMember(member.sessionId, request.prompt))

    return { ok: true, member }
  }

  #createMember(spec) {
    const sessionContext = createSessionContext({
      folderPath: this.options.folderPath,
      parentSignal: spec.parentSignal,
      swarm: this,
      bus: this.bus,
    })

    /** @type {import('./types.js').SwarmMember} */
    const member = {
      sessionId: sessionContext.sessionId,
      swarmId: this.options.swarmId,
      reportBackTo: spec.reportBackTo,
      role: spec.role,
      status: spec.status,
      friendlyName: spec.friendlyName,
      taskLabel: spec.taskLabel,
      createdAt: Date.now(),
    }

    const engine = new QueryEngine({
      tools: this.options.tools,
      getAppState: this.options.getAppState,
      setAppState: this.options.setAppState,
      systemPrompt: this.options.systemPromptFor(member),
      sessionContext,
      createStream: this.options.createStreamFactory(),
    })

    this.registry.add(member, engine, sessionContext)
    this.bus.emit('swarm:member_added', { member })
    return member
  }

  /** Runs one member's turn to completion and records its report. */
  async runMember(sessionId, prompt) {
    const engine = this.registry.engine(sessionId)
    if (!engine) return ''

    this.#setStatus(sessionId, 'running')

    try {
      let lastText = ''

      for await (const event of engine.submitMessage(prompt)) {
        this.bus.emit('swarm:agent_event', { sessionId, event })

        if (event?.type === 'assistant' && Array.isArray(event.content)) {
          const text = event.content
            .filter(b => b.type === 'text')
            .map(b => b.text)
            .join('\n')
            .trim()
          if (text) lastText = text
        }
      }

      this.registry.update(sessionId, { status: 'completed', latestReport: lastText })
      this.bus.emit('swarm:status', { sessionId, status: 'completed' })
      reportToParent(this.registry, sessionId, lastText)
      return lastText
    } catch (err) {
      this.registry.update(sessionId, {
        status: 'failed',
        latestReport: `Failed: ${err.message}`,
      })
      this.bus.emit('swarm:status', { sessionId, status: 'failed' })
      return ''
    }
  }

  #setStatus(sessionId, status) {
    this.registry.setStatus(sessionId, status)
    this.bus.emit('swarm:status', { sessionId, status })
  }

  /** Await one member's detached turn. */
  join(sessionId) {
    return this.running.get(sessionId) ?? Promise.resolve('')
  }

  /** Await every detached turn currently in flight. */
  async joinAll() {
    await Promise.allSettled([...this.running.values()])
  }
}
```

**Note what `#createMember` did not need to invent.** The session context, the
engine, the message history, the tool list - all of that is your existing
architecture. The only genuinely new field in the entire object is
`reportBackTo`.

---

# STEP 11 - `src/tools/SpawnTool/SpawnTool.js`

**The payoff for a decision you made long ago:** `sessionContext` is already
handed to every tool, and Step 10 put `swarm` on it. So the spawn tool reaches
the swarm through plumbing that already existed. No new wiring, no globals, no
dependency injection ceremony.

```javascript
// src/tools/SpawnTool/SpawnTool.js
import { z } from 'zod'
import { buildTool } from '../../Tool.js'

export const SpawnTool = buildTool({
  name: 'spawn',
  description:
    'Spawn a teammate agent to work on a sub-task in parallel. Returns immediately; ' +
    'the teammate reports back when it finishes.',

  inputSchema: z.object({
    prompt: z.string().describe('Full instructions for the new agent'),
    taskLabel: z.string().optional().describe('Short label shown in status views'),
  }),

  async call(input, context) {
    const { swarm, sessionId } = context.sessionContext ?? {}
    if (!swarm) {
      throw new Error('Swarm is not enabled for this session.')
    }

    const result = swarm.spawn({
      requesterId: sessionId,
      prompt: input.prompt,
      taskLabel: input.taskLabel,
    })

    if (!result.ok) {
      return { data: `Cannot spawn: ${result.reason}` }
    }

    // Returns now. The teammate is already running.
    return {
      data:
        `Spawned ${result.member.friendlyName} (${result.member.sessionId}). ` +
        `It is running now; its report will arrive when it finishes.`,
    }
  },

  async checkPermissions(_input, _context) {
    return { granted: true }
  },

  // Spawning is cheap, independent, and has no filesystem side effects.
  isConcurrencySafe: () => true,
  isReadOnly: () => true,
})
```

Register it in `src/tools.js` under a new `Swarm` group.

**Why the tool returns before the child finishes:** if `spawn` awaited the
child, ten spawns would run one after another and you would have built a slow
single agent with extra steps. Returning immediately is what lets the next
`spawn` overlap this one. Parallelism comes from *not waiting here*, and from
choosing deliberately *where* to wait - which is the whole of Part 2.

**A caution specific to your executor:** `isReadOnly: () => true` matters more
than it looks. `StreamingToolExecutor` reads that field to decide what may run
in parallel (Step 3). Mark spawn as not-read-only and your fan-out serialises
at the queue gate, one spawn at a time, and the swarm quietly loses its
parallelism.

---

# STEP 12 - `src/swarm/lifecycle.js`

**What jcode does:** when a member leaves mid-tree its children are
**reparented rather than orphaned** - they attach to their live grandparent,
falling back to the current coordinator, else they become roots
(docs/SWARM_ARCHITECTURE.md:40-45, implemented at
`crates/jcode-app-core/src/server/swarm.rs:1122-1156`, with coordinator
re-election just above at `:1056-1070`).

**Why this matters more than it sounds:** ownership, stop permission, broadcast
reach, and report routing are *all* derived from parent pointers. Orphan a node
and you have not merely lost a link - you have silently broken authorization,
delivery, and reporting for that entire branch.

```javascript
// src/swarm/lifecycle.js
import { childrenOf, parentOf } from './ancestry.js'
import { isTerminalStatus } from './types.js'

/**
 * Delivers a finished member's report to its parent by appending a user
 * message to the parent's QueryEngine, so the parent sees it next turn.
 */
export function reportToParent(registry, sessionId, report) {
  const member = registry.get(sessionId)
  if (!member?.reportBackTo || !report) return

  const parentEngine = registry.engine(member.reportBackTo)
  if (!parentEngine) return

  const label = member.taskLabel ? ` - ${member.taskLabel}` : ''
  parentEngine.messages = [
    ...parentEngine.messages,
    {
      type: 'user',
      content: `[report from ${member.friendlyName}${label}]\n${report}`,
    },
  ]
}

/**
 * Removes a member and reparents its children:
 *   live grandparent -> current coordinator -> root
 */
export function removeMember(registry, sessionId, status = 'stopped') {
  const member = registry.get(sessionId)
  if (!member) return

  registry.setStatus(sessionId, status)

  const grandparent = parentOf(registry, sessionId)
  const fallback = pickFallbackParent(registry, sessionId, grandparent)

  for (const child of childrenOf(registry, sessionId)) {
    registry.update(child.sessionId, { reportBackTo: fallback })
  }

  registry.remove(sessionId)
}

function pickFallbackParent(registry, leavingId, grandparent) {
  if (grandparent && !isTerminalStatus(grandparent.status)) {
    return grandparent.sessionId
  }
  const coordinator = registry.all().find(
    m => m.role === 'coordinator' && m.sessionId !== leavingId && !isTerminalStatus(m.status)
  )
  return coordinator?.sessionId ?? null   // null = becomes a root
}
```

**Checkpoint - Part 1 works.** Worth writing as a real test:

```javascript
// test/swarm.test.js
import { describe, it, expect } from 'vitest'
import { Swarm } from '../src/swarm/Swarm.js'
import { createMockClientFactory } from '../src/services/api/mockClient.js'
import { SpawnTool } from '../src/tools/SpawnTool/SpawnTool.js'
import { subtreeOf } from '../src/swarm/ancestry.js'
import { createStore } from '../src/state/store.js'
import { getDefaultAppState } from '../src/state/AppStateStore.js'

describe('spawn', () => {
  it('root fans out to three workers that all run', async () => {
    const store = createStore(getDefaultAppState())

    const swarm = new Swarm({
      swarmId: 'test',
      folderPath: process.cwd(),
      tools: [SpawnTool],
      getAppState: store.getState,
      setAppState: store.setState,
      systemPromptFor: () => 'You are a worker.',
      policy: { mode: 'light', maxLiveWorkers: 8 },
      createStreamFactory: createMockClientFactory((turn) =>
        turn === 0
          ? {
              text: 'Splitting the work.',
              toolCalls: [
                { name: 'spawn', input: { prompt: 'part A', taskLabel: 'A' } },
                { name: 'spawn', input: { prompt: 'part B', taskLabel: 'B' } },
                { name: 'spawn', input: { prompt: 'part C', taskLabel: 'C' } },
              ],
            }
          : { text: 'Done.' },
        { latencyMs: 5 }
      ),
    })

    const root = swarm.createRoot('Do a three-part job')
    await swarm.runMember(root.sessionId, 'Do a three-part job')
    await swarm.joinAll()

    expect(subtreeOf(swarm.registry, root.sessionId)).toHaveLength(3)
    expect(swarm.registry.all().every(m => m.status === 'completed')).toBe(true)
  })

  it('light mode forbids a worker from spawning', async () => {
    // Build the swarm in 'light' mode, spawn one worker, then have that worker
    // call spawn. Expect ok:false and a reason mentioning 'root'.
  })
})
```

Fill in the second test yourself - getting `canSpawn` to reject from depth 1 is
the fastest way to confirm your `depthOf` walk is right.

---
---

# PART 2 - RUNNING THEM AT ONCE

You can spawn agents. They already overlap, because `spawn` returns
immediately. What you cannot yet do is **wait well**, and waiting well is the
actual skill.

There are three ways to collect results from concurrent work. They are not
interchangeable, and picking the wrong one is the most common way a swarm ends
up slower than a single agent.

```
                   set known    react as      needs a
                   up front?    each lands?   deadline?
planFanOut         yes          no            no        Promise.all
drainAsCompleted   yes          yes           no        incremental drain
awaitMembers       no           yes           yes       event + re-check state
```

---

# STEP 13 - `src/services/swarm/planFanOut.js`

**What jcode does:** an LLM planner decomposes a task, the pieces run
concurrently under `try_join_all`, and the results are folded back into an
integration prompt (`crates/jcode-app-core/src/server/swarm.rs:1657-1671`).

**One honest caveat, because the handbook chased it down:** that specific
function in jcode is reachable only from two debug-socket commands
(`debug_command_exec.rs:132-139`, `debug_jobs.rs:77-95`), not from the live
swarm tool a production agent calls. The *shape* is real and worth learning;
just do not believe it is what fires when jcode spawns an agent.

**When to reach for it:** you know the full set of work up front, you want all
of it, and you have nothing useful to do until it is finished.

```javascript
// src/services/swarm/planFanOut.js

/**
 * Planner -> parallel workers -> integrate.
 *
 * Every worker starts before any is awaited. That ordering is the entire
 * point: build the array of promises first, then await the array.
 *
 * @param {import('../../swarm/Swarm.js').Swarm} swarm
 * @param {string} coordinatorId
 * @param {Array<{ prompt: string, taskLabel?: string }>} tasks
 * @returns {Promise<Array<{ taskLabel?: string, report: string }>>}
 */
export async function planFanOut(swarm, coordinatorId, tasks) {
  const spawned = []

  for (const task of tasks) {
    const result = swarm.spawn({
      requesterId: coordinatorId,
      prompt: task.prompt,
      taskLabel: task.taskLabel,
    })
    if (result.ok) {
      spawned.push({ taskLabel: task.taskLabel, sessionId: result.member.sessionId })
    }
  }

  // Fan in. Promise.all preserves INPUT order in its results even though the
  // promises settle in whatever order they finish.
  const reports = await Promise.all(spawned.map(s => swarm.join(s.sessionId)))

  return spawned.map((s, i) => ({ taskLabel: s.taskLabel, report: reports[i] }))
}

/** Folds worker reports into a single prompt for the coordinator's next turn. */
export function buildIntegrationPrompt(originalTask, results) {
  const sections = results.map(
    r => `### ${r.taskLabel ?? 'subtask'}\n${r.report || '(no report)'}`
  )
  return [
    `Your teammates finished their parts of: ${originalTask}`,
    '',
    ...sections,
    '',
    'Integrate these into a single coherent answer. Note any contradictions between them.',
  ].join('\n')
}
```

**The failure mode to know:** `Promise.all` rejects on the first rejection. The
other workers are *not* cancelled - they keep running, detached, and you have
lost the handle to their results. `Swarm.runMember` already catches its own
errors and returns `''`, so this stays safe here. If you ever change that,
switch to `Promise.allSettled`.

---

# STEP 14 - `src/services/swarm/drainAsCompleted.js`

**What jcode does:** `FuturesUnordered` inside the batch tool, built at
`crates/jcode-app-core/src/tool/batch.rs:282-295` and drained as results land
at `:300-317`, publishing progress after each one.

**When to reach for it:** same fixed set of work, but you want to *do something*
as each result arrives - stream it to a UI, update a progress line, stop early
once you have enough.

```javascript
// src/services/swarm/drainAsCompleted.js

/**
 * Yields results in COMPLETION order, not declaration order.
 *
 * The bookkeeping trick: race the map's values, and when one settles remove it
 * so the next race is over what remains. Racing the same settled promise
 * forever is the classic bug here.
 *
 * @template T
 * @param {Map<string, Promise<T>>} pending
 * @returns {AsyncGenerator<{ key: string, value: T }>}
 */
export async function* drainAsCompleted(pending) {
  const remaining = new Map(pending)

  while (remaining.size > 0) {
    const tagged = [...remaining.entries()].map(([key, promise]) =>
      promise.then(
        value => ({ key, value, ok: true }),
        error => ({ key, value: error, ok: false })
      )
    )

    const settled = await Promise.race(tagged)
    remaining.delete(settled.key)

    yield { key: settled.key, value: settled.value }
  }
}

/**
 * Convenience wrapper over a swarm: drain member reports as they land.
 *
 * @param {import('../../swarm/Swarm.js').Swarm} swarm
 * @param {string[]} sessionIds
 */
export async function* drainMemberReports(swarm, sessionIds) {
  const pending = new Map(sessionIds.map(id => [id, swarm.join(id)]))

  for await (const { key, value } of drainAsCompleted(pending)) {
    yield {
      sessionId: key,
      member: swarm.registry.get(key),
      report: value,
    }
  }
}
```

**Cost note worth internalising:** each pass builds a fresh array of wrapper
promises, so draining N items is O(N^2) wrappers. For a handful of swarm
workers that is irrelevant. For thousands it is not, and that is precisely why
Rust has a purpose-built `FuturesUnordered` instead of doing this.

---

# STEP 15 - `src/services/swarm/awaitMembers.js`

**What jcode does:** a long-lived task that `select!`s between a deadline timer
and a broadcast receiver, re-checking a satisfaction predicate against shared
state on every wake (`crates/jcode-app-core/src/server/comm_await.rs:271-300`).

**When to reach for it:** the set of things you are waiting on can *change while
you wait*. New members appear, some finish, some crash. There is no fixed array
of promises to hand `Promise.all`, so you wait on events instead.

**The load-bearing idea in this whole step:** when an event wakes you, **do not
trust the event payload - re-derive the answer from the registry.** Events can
be stale, duplicated, or arrive out of order. The registry is the single source
of truth. This is the discipline that makes event-driven fan-in correct, and
skipping it produces bugs that only appear under load.

```javascript
// src/services/swarm/awaitMembers.js
import { isTerminalStatus } from '../../swarm/types.js'
import { subtreeOf } from '../../swarm/ancestry.js'

/**
 * Waits until a condition over swarm members holds, or the deadline passes.
 *
 * @param {import('../../swarm/Swarm.js').Swarm} swarm
 * @param {Object} options
 * @param {string[]} [options.sessionIds]   Explicit members to watch
 * @param {string}   [options.subtreeOf]    ...or everything below this member
 * @param {'all'|'any'} [options.mode]      Default 'all'
 * @param {number}   [options.timeoutMs]    Default 30s
 * @returns {Promise<{ satisfied: boolean, timedOut: boolean, members: Array }>}
 */
export function awaitMembers(swarm, options = {}) {
  const mode = options.mode ?? 'all'
  const timeoutMs = options.timeoutMs ?? 30_000
  const deadline = Date.now() + timeoutMs

  /** Re-derived from the registry on every wake. Never from an event payload. */
  const check = () => {
    const watched = options.sessionIds
      ? options.sessionIds.map(id => swarm.registry.get(id)).filter(Boolean)
      : options.subtreeOf
        ? subtreeOf(swarm.registry, options.subtreeOf)
        : swarm.registry.all()

    if (watched.length === 0) return { satisfied: true, members: [] }

    const finished = watched.filter(m => isTerminalStatus(m.status))
    const satisfied = mode === 'all'
      ? finished.length === watched.length
      : finished.length > 0

    return { satisfied, members: watched }
  }

  return new Promise((resolve) => {
    const settle = (timedOut) => {
      clearTimeout(timer)
      unsubscribe()
      const state = check()
      resolve({ satisfied: state.satisfied, timedOut, members: state.members })
    }

    // Check once before subscribing: the condition may already hold, and
    // waiting for an event that will never come again is a deadlock.
    const initial = check()
    if (initial.satisfied) {
      return resolve({ satisfied: true, timedOut: false, members: initial.members })
    }

    const timer = setTimeout(() => settle(true), Math.max(0, deadline - Date.now()))

    const unsubscribe = swarm.bus.on('swarm:status', () => {
      // Ignore the payload entirely. Ask the registry.
      if (check().satisfied) settle(false)
    })
  })
}
```

**Read that initial `check()` again.** Subscribing and *then* checking is a
race: if the last member finished between your decision to wait and your
subscription, the event already fired and nothing will wake you. You wait the
full timeout for something that already happened. jcode's `InterruptSignal`
carries an equivalent guard for the same reason
(`crates/jcode-agent-runtime/src/lib.rs:92-106`) - it enables notification
before checking the flag, so a wakeup cannot slip through the gap.

Note also that `swarm.bus.on()` returns its unsubscribe function - your
`EventBus` was designed that way from the start, which makes this cleanup a
one-liner.

---

# STEP 16 - `src/swarm/abortTree.js`

**What jcode does:** `RuntimeTaskScope`
(`crates/jcode-app-core/src/server/runtime.rs:27-79`) pairs a `JoinSet` with a
`CancellationToken`, and its `shutdown` deliberately drains the set *before*
awaiting children, with a comment explaining that holding the lock while
joining would deadlock a task waiting to register.

**What you need:** cancellation already flows downhill, because
`createSessionContext` chains each child's controller to its parent's signal
(Step 4). So stopping a subtree is one `abort()`. What this step adds is the
other half - **signalling is not the same as waiting.**

```javascript
// src/swarm/abortTree.js
import { subtreeOf } from './ancestry.js'
import { isTerminalStatus } from './types.js'

/**
 * Aborts a member and, through the signal chain, everything beneath it.
 * Returns immediately - this signals, it does not wait.
 */
export function abortSubtree(swarm, sessionId) {
  const context = swarm.registry.context(sessionId)
  context?.abort.abort()

  for (const descendant of subtreeOf(swarm.registry, sessionId)) {
    if (!isTerminalStatus(descendant.status)) {
      swarm.registry.setStatus(descendant.sessionId, 'stopped')
    }
  }
  swarm.registry.setStatus(sessionId, 'stopped')
  swarm.bus.emit('swarm:status', { sessionId, status: 'stopped' })
}

/**
 * Signal, then wait, then give up.
 *
 * A tool mid-flight will not vanish because you called abort() - it has to
 * notice. Marking a member 'stopped' while its promise is still resolving is
 * how you get output arriving after shutdown.
 */
export async function shutdownSwarm(swarm, options = {}) {
  const graceMs = options.graceMs ?? 2_000

  for (const root of swarm.registry.all()) {
    swarm.registry.context(root.sessionId)?.abort.abort()
  }

  const inFlight = [...swarm.running.values()]

  const drained = await Promise.race([
    Promise.allSettled(inFlight).then(() => true),
    new Promise(resolve => setTimeout(() => resolve(false), graceMs)),
  ])

  if (!drained) {
    for (const member of swarm.registry.all()) {
      if (!isTerminalStatus(member.status)) {
        swarm.registry.setStatus(member.sessionId, 'crashed')
      }
    }
  }

  return { cleanShutdown: drained }
}
```

**Why stragglers become `crashed` rather than `stopped`:** `stopped` claims a
clean deliberate exit. A worker that ignored the abort signal for two seconds
did not exit cleanly, and pretending otherwise hides a real bug - usually a
tool that never wired up `signal` (which is exactly why Step 3 threaded it into
`BashTool`).

---

# STEP 17 - `src/swarm/interrupt.js`

**What jcode does:** notifications - DMs, broadcasts, lifecycle events - are
queued as soft interrupts and injected into a running agent at safe points, so
a message can be delivered mid-work without starting a new turn
(docs/SWARM_ARCHITECTURE.md, "Communication").

**The distinction that matters:** a *hard* interrupt is abort - stop now,
discard work. A *soft* interrupt is "here is some information, use it when you
next come up for air." A swarm needs both, and confusing them costs you either
responsiveness or correctness.

**What counts as a safe point here:** the boundary between turns in `query.js`.
Never mid-tool - a half-written file does not want a message - and never
mid-stream, because you would be mutating history the API call is already
using.

```javascript
// src/swarm/interrupt.js

/** @type {Map<string, string[]>} */
const queues = new Map()

/** Queue a message for delivery at the target's next safe point. */
export function queueInjection(sessionId, text) {
  const existing = queues.get(sessionId) ?? []
  queues.set(sessionId, [...existing, text])
}

/** Takes everything queued and clears it. Called by the loop between turns. */
export function drainInjections(sessionId) {
  const pending = queues.get(sessionId) ?? []
  queues.delete(sessionId)
  return pending
}

export function hasPendingInjections(sessionId) {
  return (queues.get(sessionId) ?? []).length > 0
}
```

**Wire it into the loop.** In `src/query.js`, immediately after the abort check
you added in Step 4:

```javascript
  while (turns < maxTurns) {
    if (sessionContext.signal?.aborted) {
      console.log('\n[aborted] Session cancelled, stopping loop.')
      break
    }

    // Safe point: deliver anything queued for this session before the next
    // API call, so it becomes part of the model's context naturally.
    const injections = drainInjections(sessionContext.sessionId)
    if (injections.length > 0) {
      messages = [
        ...messages,
        ...injections.map(text => ({ type: 'user', content: text })),
      ]
      onMessagesUpdate(messages)
    }

    turns++
```

**The rule that keeps this from becoming a billing incident:** a *completed*
agent does not wake up because a message arrived. It resumes only when its
owner explicitly assigns new work. jcode is emphatic about this
(docs/SWARM_ARCHITECTURE.md, "Completed or idle agents do not resume
automatically"), and the reason is arithmetic: if finished agents woke on every
broadcast, and every wake could broadcast, you have built a feedback loop that
bills you per iteration.

---
---

# PART 3 - TALKING TO EACH OTHER

Your `EventBus` doc comment says it "keeps the query loop, WS server, and UI
decoupled - each just subscribes to the event types it cares about." That is
precisely what a swarm needs, so Part 3 is mostly extension rather than
invention.

Two ideas do need care: **reach** (who hears a message) and **cost** (how much
context you pay to look at another agent).

---

# STEP 18 - `src/events/swarmEvents.js`

**Why constants rather than string literals:** you are about to emit and
subscribe to these from a dozen places. One typo in `'swarm:staus'` produces a
listener that silently never fires - no error, no warning, just a UI that never
updates.

```javascript
// src/events/swarmEvents.js

export const SwarmEvents = {
  MEMBER_ADDED: 'swarm:member_added',
  STATUS:       'swarm:status',
  AGENT_EVENT:  'swarm:agent_event',
  MESSAGE:      'swarm:message',
  FILE_TOUCH:   'swarm:file_touch',
}

/**
 * @typedef {Object} SwarmMessage
 * @property {string} fromSessionId
 * @property {string|null} toSessionId   null = broadcast
 * @property {string|null} channel       null unless it is a channel post
 * @property {'dm'|'subtree'|'channel'} scope
 * @property {string} text
 * @property {number} at
 */

/**
 * @typedef {Object} FileTouch
 * @property {string} sessionId
 * @property {string} path
 * @property {'read'|'write'} kind
 * @property {number} at
 */
```

---

# STEP 19 - `src/events/EventBus.js` - add a replay buffer

**What jcode does:** the global bus is a `tokio::sync::broadcast`
(`crates/jcode-base/src/bus.rs:499-502`), which keeps a bounded ring of recent
messages so a subscriber that falls behind is explicitly told it lagged rather
than silently missing events.

**Why you need the buffer:** in a swarm, subscribers arrive late. A member
spawned at second ten has missed everything before it. A UI panel opened
mid-run shows an empty screen. Your current bus emits synchronously to whoever
is listening *right now*, and anything earlier is gone forever.

**The honest limitation to know:** `EventEmitter`-style buses cannot detect a
slow consumer the way Rust's `broadcast` can. Rust hands a lagging receiver a
`Lagged(n)` error saying exactly how many messages it missed. You just... miss
them. A ring buffer narrows the window; it does not close it.

```javascript
// src/events/EventBus.js

/**
 * In-process pub/sub bus.
 * Keeps the query loop, WS server, and UI decoupled - each just
 * subscribes to the event types it cares about.
 *
 * Usage:
 *   const bus = new EventBus()
 *   const unsub = bus.on('event', (e) => console.log(e))
 *   bus.emit('event', { ... })
 *   unsub()
 *
 * Swarm addition: a bounded replay buffer, so a subscriber that attaches late
 * can catch up. jcode gets this from tokio::sync::broadcast's ring
 * (crates/jcode-base/src/bus.rs:499-502).
 */
export class EventBus {
  constructor(options = {}) {
    /** @type {Map<string, Set<Function>>} */
    this.listeners = new Map()
    this.historyLimit = options.historyLimit ?? 500
    /** @type {Array<{ type: string, payload: any, at: number }>} */
    this.history = []
  }

  /**
   * Subscribe to an event type.
   * @returns {() => void} unsubscribe function
   */
  on(type, fn) {
    if (!this.listeners.has(type)) this.listeners.set(type, new Set())
    this.listeners.get(type).add(fn)
    return () => this.listeners.get(type)?.delete(fn)
  }

  /** Synchronously notify all subscribers of `type`. */
  emit(type, payload) {
    this.history.push({ type, payload, at: Date.now() })
    if (this.history.length > this.historyLimit) {
      this.history.splice(0, this.history.length - this.historyLimit)
    }

    for (const fn of this.listeners.get(type) ?? []) {
      try {
        fn(payload)
      } catch (err) {
        // One broken subscriber must not take down the emitter. In a swarm
        // the emitter is usually an agent finishing its turn.
        console.error(`[EventBus] listener for '${type}' threw:`, err.message)
      }
    }
  }

  /** Replay recent events, optionally of one type. For late subscribers. */
  replay(type = null, limit = 100) {
    const matching = type
      ? this.history.filter(e => e.type === type)
      : this.history
    return matching.slice(-limit)
  }
}
```

**The `try/catch` is not defensive padding.** Your original `emit` calls
listeners directly, so a UI component that throws would propagate the exception
up into whichever agent happened to emit - killing that agent's turn for a
reason that has nothing to do with its work. With many agents emitting
constantly, that goes from unlikely to routine.

---

# STEP 20 - `src/services/comms/routing.js`

**What jcode does:** one send path routes by which fields are populated - with
`to_session` it is a DM, with `channel` it posts to that channel, with neither
it broadcasts to **the sender's spawned subtree**
(`crates/jcode-app-core/src/server/client_comm_message.rs:253-269`).

**The design decision worth stealing:** the default reach of a broadcast is
your own subtree, *not* the whole swarm. Whole-swarm reach is an escape hatch
reserved for the coordinator. In a twenty-agent swarm, "tell everyone" is
almost always wrong - it is twenty context injections, nineteen of them
irrelevant, all of them billed.

```javascript
// src/services/comms/routing.js
import { SwarmEvents } from '../../events/swarmEvents.js'
import { subtreeOf } from '../../swarm/ancestry.js'
import { isTerminalStatus } from '../../swarm/types.js'
import { queueInjection } from '../../swarm/interrupt.js'

/**
 * Routes by which fields are present:
 *   toSessionId -> DM
 *   channel     -> channel post
 *   neither     -> broadcast to the sender's own subtree
 *
 * @returns {{ delivered: number, scope: string }}
 */
export function send(swarm, fromSessionId, options) {
  const { toSessionId = null, channel = null, text } = options

  const scope = toSessionId ? 'dm' : channel ? 'channel' : 'subtree'
  const targets = resolveTargets(swarm, fromSessionId, { toSessionId, channel })

  const sender = swarm.registry.get(fromSessionId)
  const label = sender?.friendlyName ?? fromSessionId

  let delivered = 0
  for (const target of targets) {
    // Completed agents do not wake for messages (see Step 17).
    if (isTerminalStatus(target.status)) continue

    queueInjection(target.sessionId, `[${scope} from ${label}]\n${text}`)
    delivered++
  }

  swarm.bus.emit(SwarmEvents.MESSAGE, {
    fromSessionId, toSessionId, channel, scope, text, at: Date.now(),
  })

  return { delivered, scope }
}

function resolveTargets(swarm, fromSessionId, { toSessionId, channel }) {
  if (toSessionId) {
    const target = swarm.registry.get(toSessionId)
    return target ? [target] : []
  }

  if (channel) {
    const members = swarm.channels?.get(channel) ?? new Set()
    return [...members]
      .filter(id => id !== fromSessionId)
      .map(id => swarm.registry.get(id))
      .filter(Boolean)
  }

  // Default: your own subtree. Not the whole swarm.
  return subtreeOf(swarm.registry, fromSessionId)
}

/** Coordinator-only escape hatch. */
export function broadcastAll(swarm, fromSessionId, text) {
  const sender = swarm.registry.get(fromSessionId)
  if (sender?.role !== 'coordinator') {
    return { ok: false, reason: 'Only the coordinator may broadcast to the whole swarm' }
  }

  let delivered = 0
  for (const member of swarm.registry.all()) {
    if (member.sessionId === fromSessionId) continue
    if (isTerminalStatus(member.status)) continue
    queueInjection(member.sessionId, `[swarm broadcast]\n${text}`)
    delivered++
  }

  return { ok: true, delivered }
}

export function joinChannel(swarm, sessionId, channel) {
  swarm.channels ??= new Map()
  if (!swarm.channels.has(channel)) swarm.channels.set(channel, new Set())
  swarm.channels.get(channel).add(sessionId)
}
```

**The message tool** is the same shape as `SpawnTool` - it reaches the swarm
through `sessionContext`:

```javascript
// src/tools/MessageTool/MessageTool.js
import { z } from 'zod'
import { buildTool } from '../../Tool.js'
import { send } from '../../services/comms/routing.js'

export const MessageTool = buildTool({
  name: 'message',
  description:
    'Send a message to a teammate. With to_session it is a direct message; ' +
    'with neither to_session nor channel it goes to the agents you spawned.',

  inputSchema: z.object({
    text: z.string(),
    to_session: z.string().optional(),
    channel: z.string().optional(),
  }),

  async call(input, context) {
    const { swarm, sessionId } = context.sessionContext ?? {}
    if (!swarm) throw new Error('Swarm is not enabled for this session.')

    const result = send(swarm, sessionId, {
      text: input.text,
      toSessionId: input.to_session ?? null,
      channel: input.channel ?? null,
    })

    return { data: `Delivered to ${result.delivered} agent(s) via ${result.scope}.` }
  },

  async checkPermissions() { return { granted: true } },
  isConcurrencySafe: () => true,
  isReadOnly: () => true,
})
```

---

# STEP 21 - `src/services/comms/reads.js`

**What jcode does:** three genuinely separate operations, in
`crates/jcode-app-core/src/server/comm_sync.rs` - status, summary, and full
context read.

**Why three and not one:** they cost wildly different amounts of context, and
collapsing them means every glance costs the maximum.

```
statusSnapshot   ~50 tokens     "worker-3 is running, on 'display detection'"
activitySummary  ~500 tokens    the tool calls it made and short results
fullContext      ~50,000 tokens its entire transcript
```

**The property that must hold:** a status snapshot has to work *while the target
is busy*. If checking on an agent requires waiting for it to become idle, your
coordinator blocks on exactly the agents it most needs to watch. Snapshot reads
only the registry - which is why the registry is the source of truth and the
transcript is not.

```javascript
// src/services/comms/reads.js
import { isInSubtree } from '../../swarm/ancestry.js'

/** Tier 1: cheap, always available, safe to poll. */
export function statusSnapshot(swarm, sessionId) {
  const member = swarm.registry.get(sessionId)
  if (!member) return null

  return {
    sessionId: member.sessionId,
    friendlyName: member.friendlyName,
    status: member.status,
    taskLabel: member.taskLabel,
    role: member.role,
    reportBackTo: member.reportBackTo,
    ageMs: Date.now() - member.createdAt,
  }
}

/** Tier 2: recent activity, derived from the bus rather than the transcript. */
export function activitySummary(swarm, sessionId, limit = 10) {
  const events = swarm.bus
    .replay('swarm:agent_event', 500)
    .filter(e => e.payload.sessionId === sessionId)
    .slice(-limit)

  const lines = events.map(({ payload }) => {
    const event = payload.event
    if (event?.type === 'tool_result') {
      return `  result: ${String(event.content).slice(0, 100)}`
    }
    if (event?.type === 'assistant' && Array.isArray(event.content)) {
      const calls = event.content.filter(b => b.type === 'tool_use').map(b => b.name)
      if (calls.length > 0) return `  calling: ${calls.join(', ')}`
      const text = event.content.filter(b => b.type === 'text').map(b => b.text).join(' ')
      return text ? `  said: ${text.slice(0, 100)}` : null
    }
    return null
  }).filter(Boolean)

  return {
    ...statusSnapshot(swarm, sessionId),
    recent: lines,
  }
}

/**
 * Tier 3: the whole transcript. Expensive, and authorization-gated.
 *
 * jcode restricts this to yourself or your own subtree; a coordinator may read
 * anyone. Reading a peer's full context is how one agent's 50K tokens silently
 * becomes part of another agent's bill.
 */
export function fullContext(swarm, requesterId, targetId) {
  const requester = swarm.registry.get(requesterId)
  const allowed =
    requesterId === targetId ||
    requester?.role === 'coordinator' ||
    isInSubtree(swarm.registry, requesterId, targetId)

  if (!allowed) {
    return { ok: false, reason: 'You may only read your own context or an agent you spawned.' }
  }

  const engine = swarm.registry.engine(targetId)
  if (!engine) return { ok: false, reason: `No such member: ${targetId}` }

  return { ok: true, messages: engine.getMessages() }
}
```

---

# STEP 22 - `src/services/comms/persist.js`

**What jcode does:** members are saved as durable records and, on load,
`recover_member_status` **rewrites** the status rather than restoring it -
`Running` becomes `Crashed`, `Ready` becomes `Stopped`
(`crates/jcode-app-core/src/server/swarm_persistence.rs:341-390`).

**Why rewriting is correct, and restoring is a bug:** "running" describes a live
process. That process died with the daemon. Restoring the string faithfully
would give you a registry full of agents that appear to be working and never
will be - and a coordinator that waits forever for reports from ghosts. The
comment in jcode's source about resurrecting hundreds of detached historical
clients is the scar tissue from learning this the hard way.

```javascript
// src/services/comms/persist.js
import fs from 'node:fs/promises'

/** Members only. Engines hold live sockets and promises; those do not persist. */
export async function saveSnapshot(swarm, filePath) {
  const snapshot = {
    swarmId: swarm.options.swarmId,
    savedAt: Date.now(),
    members: swarm.registry.all(),
  }
  await fs.writeFile(filePath, JSON.stringify(snapshot, null, 2), 'utf8')
  return snapshot
}

/**
 * Statuses that described a live process cannot survive a restart.
 * This mirrors jcode's recover_member_status.
 */
export function recoverStatus(status) {
  switch (status) {
    case 'running':
    case 'spawned':
    case 'blocked':
      return 'crashed'    // it was alive; it is not any more
    case 'ready':
      return 'stopped'    // it was idle and waiting; nothing is waiting now
    default:
      return status       // completed / failed / stopped / crashed are already final
  }
}

export async function loadSnapshot(filePath) {
  const raw = await fs.readFile(filePath, 'utf8')
  const snapshot = JSON.parse(raw)

  return {
    ...snapshot,
    members: snapshot.members.map(m => ({ ...m, status: recoverStatus(m.status) })),
  }
}
```

**Checkpoint - Part 3 works.** A test that catches the subtle one:

```javascript
import { describe, it, expect } from 'vitest'
import { recoverStatus } from '../src/services/comms/persist.js'

describe('recoverStatus', () => {
  it('never restores a status that described a live process', () => {
    expect(recoverStatus('running')).toBe('crashed')
    expect(recoverStatus('ready')).toBe('stopped')
    expect(recoverStatus('completed')).toBe('completed')
  })
})
```

---
---

# PART 4 - THE TASK DAG

Everything so far made agents run in parallel. This part makes the *work*
structured - and adds the one mechanism that separates jcode from a task list
with threads: **reviewer nodes that cannot rubber-stamp.**

This is the part with no equivalent anywhere in data-swarm-cli, so all the
naming here is jcode's own. It maps directly onto
`crates/jcode-plan/src/dag/`, which is under 1,700 lines and well worth reading
once you have written your version.

**The problem being solved.** Let an agent decompose a big task into an
unbounded tree of sub-tasks, run them in parallel, and two things go wrong:

1. **The tree breaks structurally** - cycles, references to work that does not
   exist, two workers editing the same node.
2. **The tree goes shallow** - an agent says "done" without having explored
   what it was given, and nothing forces it to admit what it skipped.

jcode's answer is one data structure - a DAG of task nodes - plus a small
closed set of **validated mutations** that are the only legal way to change it.

---

# STEP 23 - `src/dag/types.js`

**What jcode does:** `crates/jcode-plan/src/dag/mod.rs` - `Mode` at `:37-49`,
`NodeOrigin` at `:58-67`, `NodeKind` and `gate_kind()` at `:72-102`.

**Two ideas here carry the whole part.**

**`NodeOrigin` measures whether thinking happened.** A node is `seed` (the first
draft), `expand` (born from decomposition), `gap` (injected by a gate that
found a hole), or `gate` (an auto-inserted reviewer). If a finished run is
still all `seed` nodes, nothing decomposed and no gate found anything - the
plan never outgrew its first guess, which is visible *structurally* rather than
by reading the output.

**`gateKind()` encodes what "checked" means for different work.** Code-shaped
work gets a `verify` gate: does it actually run? Research-shaped work gets a
`critique` gate: what did you miss? Those are different questions and conflating
them gives you reviewers that test prose and proofread code.

```javascript
// src/dag/types.js

/**
 * One engine, two presets.
 *   deep  - composite nodes get an auto-inserted gate before they can close,
 *           and completion artifacts are strictly validated
 *   light - cheap parallelism, no mandatory gates
 * The data model and scheduler are identical; only rigor changes.
 * jcode: dag/mod.rs:37-49
 *
 * @typedef {'deep'|'light'} Mode
 */

/** @param {Mode} mode */
export function requiresGates(mode) {
  return mode === 'deep'
}

/**
 * Where a node came from - the growth signal.
 * jcode: dag/mod.rs:58-67
 *
 * @typedef {'seed'|'expand'|'gap'|'gate'} NodeOrigin
 */

/**
 * The terminal action a node represents.
 * jcode: dag/mod.rs:72-102
 *
 * @typedef {'explore'|'implement'|'verify'|'fix'|'synthesize'|'critique'} NodeKind
 */

/** Gates are auto-inserted reviewers, not user-seeded work. */
export function isGateKind(kind) {
  return kind === 'critique' || kind === 'verify'
}

/**
 * Which gate guards a composite node of this kind.
 * Code-shaped work is verified; everything else is critiqued.
 */
export function gateKind(kind) {
  return kind === 'implement' || kind === 'fix' ? 'verify' : 'critique'
}

/**
 * Node lifecycle. Note there is no 'blocked' - it is COMPUTED from dependency
 * state by the scheduler, so there is exactly one source of truth.
 *
 * @typedef {'queued'|'running'|'done'|'failed'} NodeStatus
 */

/**
 * What a worker hands back when it finishes a node.
 *
 * `whatIDidNotCheck` is the important field. Most systems have nowhere to put
 * "I ran out of time before looking at the websocket path", so that knowledge
 * evaporates. Here Step 28 turns each entry into a real node.
 *
 * @typedef {Object} Artifact
 * @property {string} findings
 * @property {string} confidence          Free text on the wire; parsed in Step 24
 * @property {string[]} [whatIDidNotCheck]
 * @property {string[]} [openQuestions]
 * @property {string} [validation]        Required for implement/fix in deep mode
 */

/**
 * @typedef {Object} TaskNode
 * @property {string} id
 * @property {NodeKind} kind
 * @property {NodeOrigin} origin
 * @property {NodeStatus} status
 * @property {string} title
 * @property {string} scope
 * @property {string[]} dependsOn
 * @property {string|null} parent
 * @property {boolean} isGate
 * @property {string|null} owner
 * @property {Artifact|null} output
 */

/**
 * @typedef {Object} NodeSpec
 * @property {string} id
 * @property {NodeKind} kind
 * @property {string} title
 * @property {string} scope
 * @property {string[]} [dependsOn]
 */

/**
 * Result helpers. Every mutation returns one of these instead of throwing,
 * because the error text is read by a model and has to tell it what to do next.
 *
 * @typedef {{ ok: true, value: any } | { ok: false, error: { code: string, message: string } }} DagResult
 */

export function ok(value) {
  return { ok: true, value }
}

export function err(code, message) {
  return { ok: false, error: { code, message } }
}
```

---

# STEP 24 - `src/dag/confidence.js`

**What jcode does:** `ConfidenceLevel::parse` at
`crates/jcode-plan/src/dag/mod.rs:141-204`.

**Why parsing is lenient:** agents write confidence as free text - "high",
"medium-high", "not very sure", "7/10", "0.9". Rejecting anything unrecognised
would mean rejecting completions for formatting, so the parser accepts what
models actually emit.

**The ordering detail that is a genuine bug if you get it wrong:** check
negations *before* word rungs. "not confident" contains the substring
"confident". Match rungs first and that phrase reads as **high** confidence -
the exact inverse of what the agent said, silently erasing a debt the gate in
Step 28 exists to catch.

```javascript
// src/dag/confidence.js

/** @typedef {'low'|'medium'|'high'} ConfidenceLevel */

const NEGATIONS = [
  'not high', 'not confident', 'not certain', 'not sure',
  'no confidence', 'unsure', 'uncertain',
]

/**
 * Lenient parse of a free-text confidence field.
 * @returns {ConfidenceLevel|null} null when nothing recognisable is present
 */
export function parseConfidence(raw) {
  if (!raw || typeof raw !== 'string') return null
  const text = raw.trim().toLowerCase()
  if (text === '') return null

  // Negations FIRST. "not confident" must not match "confident" -> high.
  if (NEGATIONS.some(n => text.includes(n))) return 'low'

  // Word rungs, low before high so hedges resolve pessimistically.
  if (text.includes('low')) return 'low'
  if (text.includes('medium') || text.includes('moderate')) return 'medium'
  if (text.includes('high') || text.includes('confident') || text.includes('certain')) return 'high'

  const numeric = parseNumericConfidence(text)
  if (numeric !== null) return numeric

  return null
}

/** Accepts 0-1, 0-10, percentages, and "7 out of 10" / "1/10". */
function parseNumericConfidence(text) {
  const outOf = text.match(/(\d+(?:\.\d+)?)\s*(?:\/|out of)\s*(\d+(?:\.\d+)?)/)
  if (outOf) {
    const denominator = parseFloat(outOf[2])
    if (denominator > 0) return fromFraction(parseFloat(outOf[1]) / denominator)
  }

  const percent = text.match(/(\d+(?:\.\d+)?)\s*%/)
  if (percent) return fromFraction(parseFloat(percent[1]) / 100)

  const bare = text.match(/(\d+(?:\.\d+)?)/)
  if (bare) {
    const value = parseFloat(bare[1])
    if (value <= 1)   return fromFraction(value)
    if (value <= 10)  return fromFraction(value / 10)
    if (value <= 100) return fromFraction(value / 100)
  }

  return null
}

function fromFraction(value) {
  if (value < 0.4) return 'low'
  if (value < 0.75) return 'medium'
  return 'high'
}

/** Unparseable counts as low - silence is not confidence. */
export function isLowConfidence(raw) {
  return (parseConfidence(raw) ?? 'low') === 'low'
}

export function isHighConfidence(raw) {
  return parseConfidence(raw) === 'high'
}
```

**Note the default in `isLowConfidence`.** A missing or garbled confidence field
is treated as *low*, not as "unknown, let it pass". If an agent cannot state
how sure it is, the gate should look harder, not less hard.

---

# STEP 25 - `src/dag/TaskGraph.js`

**What jcode does:** the node collection is a **private** field
(`crates/jcode-plan/src/dag/mod.rs:541`), so the only way to change the graph is
through the validated operations in `ops.rs`.

**Why privacy is the whole design:** if agents could push nodes directly, every
invariant - no cycles, no duplicate ids, no orphans, no closing a parent with
open children - would be enforced by hoping. Making the collection private turns
those from conventions into guarantees.

**Think of it as a REST API that never lets a client write straight to a
table.** Every write goes through a handler that can say no.

```javascript
// src/dag/TaskGraph.js

/**
 * The task graph. `#nodes` is private on purpose: mutations live in ops.js,
 * complete.js and gates.js, and they are the only legal way in.
 * jcode: dag/mod.rs:541
 */
export class TaskGraph {
  #nodes

  /** @param {import('./types.js').Mode} mode */
  constructor(mode = 'deep') {
    this.mode = mode
    /** @type {Map<string, import('./types.js').TaskNode>} */
    this.#nodes = new Map()
  }

  get(id) {
    return this.#nodes.get(id)
  }

  has(id) {
    return this.#nodes.has(id)
  }

  all() {
    return [...this.#nodes.values()]
  }

  childrenOf(id) {
    return this.all().filter(n => n.parent === id)
  }

  size() {
    return this.#nodes.size
  }

  /**
   * @internal Used by the ops modules and the runner. Not for agents.
   */
  insert(node) {
    this.#nodes.set(node.id, node)
  }

  /**
   * @internal
   */
  patch(id, changes) {
    const existing = this.#nodes.get(id)
    if (!existing) return
    this.#nodes.set(id, { ...existing, ...changes })
  }

  /** A working copy. Mutations stage here, then commit atomically if valid. */
  clone() {
    const copy = new TaskGraph(this.mode)
    for (const node of this.all()) {
      copy.insert({ ...node, dependsOn: [...node.dependsOn] })
    }
    return copy
  }

  /** Seeded vs grown - the fastest read on whether the plan developed. */
  growthStats() {
    const nodes = this.all()
    return {
      seeded: nodes.filter(n => n.origin === 'seed').length,
      grown: nodes.filter(n => n.origin !== 'seed').length,
    }
  }
}

/** Depth-first cycle detection over dependsOn edges. */
export function wouldCycle(graph) {
  const state = new Map()   // id -> 'visiting' | 'done'

  const visit = (id) => {
    const current = state.get(id)
    if (current === 'visiting') return true
    if (current === 'done') return false

    state.set(id, 'visiting')
    for (const dep of graph.get(id)?.dependsOn ?? []) {
      if (graph.has(dep) && visit(dep)) return true
    }
    state.set(id, 'done')
    return false
  }

  return graph.all().some(node => visit(node.id))
}
```

**`clone()` plus `wouldCycle()` is the stage-then-commit pattern.** Every
mutation builds a candidate graph, validates it whole, and only then replaces
the live one. A rejected `expandNode` leaves the real graph untouched - no
partial writes, no half-applied decomposition to clean up.

---

# STEP 26 - `src/dag/ops.js`

**What jcode does:** `seed` at `crates/jcode-plan/src/dag/ops.rs:19-76`,
`ensure_root_gate` at `:143-209`, `expand_node` at `:227-367`.

**`expandNode` is how a worker says "this is bigger than one task".** It turns
a node into a parent of new children, and - in deep mode - the parent now waits
on a gate that waits on those children.

```javascript
// src/dag/ops.js
import { TaskGraph, wouldCycle } from './TaskGraph.js'
import { gateKind, isGateKind, requiresGates, ok, err } from './types.js'

/** Copies a staged graph over the live one. Exported - gates.js uses it too. */
export function commitStaged(target, staged) {
  for (const node of staged.all()) {
    target.insert(node)
  }
}

function specToNode(spec, parent, origin) {
  return {
    id: spec.id,
    kind: spec.kind,
    origin,
    status: 'queued',
    title: spec.title,
    scope: spec.scope,
    dependsOn: spec.dependsOn ? [...spec.dependsOn] : [],
    parent,
    isGate: isGateKind(spec.kind),
    owner: null,
    output: null,
  }
}

/**
 * Lays down the initial plan and, in deep mode, the root gate.
 * @param {TaskGraph} graph
 * @param {import('./types.js').NodeSpec[]} specs
 */
export function seed(graph, specs) {
  if (specs.length === 0) return err('invalid_spec', 'Seed requires at least one node')

  const staged = graph.clone()

  for (const spec of specs) {
    if (staged.has(spec.id)) {
      return err('duplicate_id', `Node id already exists: ${spec.id}`)
    }
    staged.insert(specToNode(spec, null, 'seed'))
  }

  for (const node of staged.all()) {
    for (const dep of node.dependsOn) {
      if (!staged.has(dep)) {
        return err('unknown_dependency', `Node ${node.id} depends on missing node ${dep}`)
      }
    }
  }

  if (wouldCycle(staged)) {
    return err('cycle', 'Those dependencies form a cycle')
  }

  if (requiresGates(staged.mode)) {
    ensureRootGate(staged)
  }

  commitStaged(graph, staged)
  return ok(undefined)
}

/**
 * In deep mode the whole plan is guarded by one root gate that depends on
 * every top-level node. Nothing finishes until a reviewer has looked.
 * jcode: ops.rs:143-209
 */
export function ensureRootGate(graph) {
  const gateId = 'root::gate'
  if (graph.has(gateId)) {
    graph.patch(gateId, {
      dependsOn: graph.all()
        .filter(n => n.parent === null && !n.isGate)
        .map(n => n.id),
    })
    return gateId
  }

  const topLevel = graph.all().filter(n => n.parent === null && !n.isGate)

  graph.insert({
    id: gateId,
    kind: 'critique',
    origin: 'gate',
    status: 'queued',
    title: 'Review the whole plan',
    scope:
      'Audit every top-level node by id. Confirm the plan actually covers the ' +
      'task, and inject work for anything missing.',
    dependsOn: topLevel.map(n => n.id),
    parent: null,
    isGate: true,
    owner: null,
    output: null,
  })

  return gateId
}

/**
 * Decomposes a node into children. The node becomes a parent that waits on
 * them (via its gate in deep mode).
 * jcode: ops.rs:227-367
 */
export function expandNode(graph, nodeId, actor, specs) {
  const node = graph.get(nodeId)
  if (!node) return err('unknown_node', `No such node: ${nodeId}`)
  if (node.isGate) {
    return err('wrong_status', `Node ${nodeId} is a gate; gates inject work, they do not expand`)
  }
  if (node.status === 'done') return err('wrong_status', `Node ${nodeId} is already done`)
  if (node.owner !== null && node.owner !== actor) {
    return err('not_owner', `Node ${nodeId} is owned by ${node.owner}, not ${actor}`)
  }
  if (specs.length === 0) {
    return err('invalid_spec', 'Expanding requires at least one child')
  }

  const staged = graph.clone()
  const childIds = []

  for (const spec of specs) {
    if (staged.has(spec.id)) {
      return err('duplicate_id', `Node id already exists: ${spec.id}`)
    }
    staged.insert(specToNode(spec, nodeId, 'expand'))
    childIds.push(spec.id)
  }

  for (const id of childIds) {
    for (const dep of staged.get(id).dependsOn) {
      if (!staged.has(dep)) {
        return err('unknown_dependency', `Child ${id} depends on missing node ${dep}`)
      }
    }
  }

  if (requiresGates(staged.mode)) {
    // The parent waits on a gate; the gate waits on the children.
    const gateId = `${nodeId}::gate`
    staged.insert({
      id: gateId,
      kind: gateKind(node.kind),
      origin: 'gate',
      status: 'queued',
      title: `Review ${node.title}`,
      scope:
        `Audit every child of ${nodeId} by id. Address each one, or inject work ` +
        `for what is missing.`,
      dependsOn: childIds,
      parent: nodeId,
      isGate: true,
      owner: null,
      output: null,
    })
    staged.patch(nodeId, { dependsOn: [...new Set([...node.dependsOn, gateId])] })
  } else {
    staged.patch(nodeId, { dependsOn: [...new Set([...node.dependsOn, ...childIds])] })
  }

  // The parent goes back in the queue: its job is now to synthesize.
  staged.patch(nodeId, { status: 'queued', owner: null })

  if (wouldCycle(staged)) {
    return err('cycle', 'That decomposition would create a cycle')
  }

  commitStaged(graph, staged)
  return ok(childIds)
}
```

---

# STEP 27 - `src/dag/complete.js`

**What jcode does:** `complete_node` at
`crates/jcode-plan/src/dag/ops.rs:377-411`, artifact validation at `:703-758`.

**The design point:** completing a node is not setting a boolean. It requires a
structured artifact, and in deep mode that artifact is validated before the
node closes. "Done" has to be earned.

```javascript
// src/dag/complete.js
import { parseConfidence } from './confidence.js'
import { requiresGates, ok, err } from './types.js'

/** Deep mode demands a substantive artifact. Light mode takes what it gets. */
export function validateArtifact(graph, node, artifact) {
  if (!requiresGates(graph.mode)) return ok(undefined)

  if (!artifact?.findings || artifact.findings.trim().length < 20) {
    return err(
      'invalid_artifact',
      `Node ${node.id}: 'findings' must actually describe what you found ` +
      `(at least a sentence). A bare "done" is not a completion report.`
    )
  }

  if (!artifact.confidence || parseConfidence(artifact.confidence) === null) {
    return err(
      'invalid_artifact',
      `Node ${node.id}: 'confidence' is required and must be readable ` +
      `(low / medium / high, or a score). You wrote: ${artifact.confidence ?? '(nothing)'}`
    )
  }

  if ((node.kind === 'implement' || node.kind === 'fix') && !artifact.validation) {
    return err(
      'invalid_artifact',
      `Node ${node.id} changed code, so 'validation' is required: ` +
      `what did you run, and what did it say?`
    )
  }

  return ok(undefined)
}

export function completeNode(graph, nodeId, actor, artifact) {
  const node = graph.get(nodeId)
  if (!node) return err('unknown_node', `No such node: ${nodeId}`)
  if (node.status === 'done') return err('wrong_status', `Node ${nodeId} is already done`)
  if (node.owner !== null && node.owner !== actor) {
    return err('not_owner', `Node ${nodeId} is owned by ${node.owner}, not ${actor}`)
  }

  const openChildren = graph.childrenOf(nodeId).filter(c => c.status !== 'done')
  if (openChildren.length > 0) {
    return err(
      'wrong_status',
      `Node ${nodeId} has unfinished children: ${openChildren.map(c => c.id).join(', ')}`
    )
  }

  const valid = validateArtifact(graph, node, artifact)
  if (!valid.ok) return valid

  graph.patch(nodeId, { status: 'done', output: artifact })
  return ok(undefined)
}

export function failNode(graph, nodeId, actor, reason) {
  const node = graph.get(nodeId)
  if (!node) return err('unknown_node', `No such node: ${nodeId}`)
  if (node.owner !== null && node.owner !== actor) {
    return err('not_owner', `Node ${nodeId} is owned by ${node.owner}, not ${actor}`)
  }

  graph.patch(nodeId, {
    status: 'failed',
    output: { findings: `FAILED: ${reason}`, confidence: 'low' },
  })
  return ok(undefined)
}

/** Put a failed node back in the queue, e.g. after a fix landed. */
export function requeueFailed(graph, nodeId) {
  const node = graph.get(nodeId)
  if (!node) return err('unknown_node', `No such node: ${nodeId}`)
  if (node.status !== 'failed') {
    return err('wrong_status', `Node ${nodeId} is ${node.status}, not failed`)
  }
  graph.patch(nodeId, { status: 'queued', owner: null, output: null })
  return ok(undefined)
}
```

---

# STEP 28 - `src/dag/gates.js` - the anti-rubber-stamp machinery

This is the step. Everything else in Part 4 exists to make it possible.

**What jcode does:** `validate_gate_pass` at
`crates/jcode-plan/src/dag/ops.rs:789-878`, `gate_audit_scope` at `:759-765`,
`mentions_node_id` at `:593-628`, `inject_from_gate` at `:444-540`.

**The problem:** a reviewer agent looks at eight finished nodes and writes "All
good, no gaps found." Did it read all eight? Did it read any? A reviewer that
can pass without demonstrating coverage is decoration.

**Three checks. A gate may only pass if:**

1. **No stale scope.** Every node it audits is actually `done`. A gate that
   started before its scope finished was auditing a moving target - reject and
   re-run later.
2. **No confidence debt.** Any audited node that self-reported *low* confidence
   must be addressed by id in `findings` or `openQuestions`.
3. **No coverage debt.** Up to 20 audited nodes, the gate must name **every**
   one - not just the shaky ones. Above 20, enumeration relaxes for
   high-confidence nodes only, so anything medium, low, or unreadable still has
   to be named.

**And the rule that closes the loophole:** the gate's own `whatIDidNotCheck`
does **not** count as addressing anything. Saying "I did not check node-7" is
the opposite of auditing node-7. Without this, a reviewer satisfies every check
by listing all its siblings as things it skipped.

```javascript
// src/dag/gates.js
import { wouldCycle } from './TaskGraph.js'
import { isHighConfidence, isLowConfidence } from './confidence.js'
import { completeNode } from './complete.js'
import { commitStaged } from './ops.js'
import { ok, err } from './types.js'

/** Above this many audited nodes, full enumeration relaxes. jcode uses 20. */
export const GATE_COVERAGE_ENUMERATION_CAP = 20

/** What a gate is responsible for: its dependencies, minus other gates. */
export function gateAuditScope(graph, gate) {
  return gate.dependsOn
    .map(id => graph.get(id))
    .filter(n => n !== undefined && !n.isGate)
}

const ID_CHAR = /[A-Za-z0-9\-_.:]/

/**
 * Whole-token match. "node-a" must not match inside "node-ab", or a lazy
 * reviewer gets credit for coverage it never provided.
 * jcode: ops.rs:593-628
 */
export function mentionsNodeId(text, id) {
  if (!text || !id) return false

  let from = 0
  while (from <= text.length) {
    const at = text.indexOf(id, from)
    if (at === -1) return false

    const before = at === 0 ? '' : text[at - 1]
    const afterIndex = at + id.length
    const after = afterIndex < text.length ? text[afterIndex] : ''

    const beforeOk = before === '' || !ID_CHAR.test(before)

    let afterOk
    if (after === '') {
      afterOk = true
    } else if (after === '.' || after === ':') {
      // Ambiguous: legal id characters AND sentence punctuation.
      // "node-a." ending a sentence is a mention; "node-a.b" is another id.
      const next = afterIndex + 1 < text.length ? text[afterIndex + 1] : ''
      afterOk = next === '' || !ID_CHAR.test(next)
    } else {
      afterOk = !ID_CHAR.test(after)
    }

    if (beforeOk && afterOk) return true
    from = at + 1
  }
  return false
}

/** Addressed = named in findings or openQuestions. whatIDidNotCheck does NOT count. */
function addressedBy(artifact, id) {
  if (mentionsNodeId(artifact?.findings ?? '', id)) return true
  return (artifact?.openQuestions ?? []).some(q => mentionsNodeId(q, id))
}

export function validateGatePass(graph, gateId, artifact) {
  const gate = graph.get(gateId)
  if (!gate) return ok(undefined)

  const scope = gateAuditScope(graph, gate)
  if (scope.length === 0) return ok(undefined)

  // 1. Stale scope.
  const pending = scope.filter(n => n.status !== 'done')
  if (pending.length > 0) {
    return err(
      'stale_gate_scope',
      `Gate ${gateId} cannot pass: these nodes are not done yet: ` +
      `${pending.map(n => n.id).join(', ')}. It will re-run once they finish.`
    )
  }

  // 2. Confidence debt - at any scope width.
  const debts = scope
    .filter(n => isLowConfidence(n.output?.confidence))
    .filter(n => !addressedBy(artifact, n.id))
  if (debts.length > 0) {
    return err(
      'unaddressed_low_confidence',
      `Gate ${gateId} cannot pass: ${debts.map(n => n.id).join(', ')} reported LOW ` +
      `confidence and you did not address them by id. Either examine them in ` +
      `findings/openQuestions, or inject a gap node to cover them.`
    )
  }

  // 3. Coverage debt.
  const mustAddress =
    scope.length <= GATE_COVERAGE_ENUMERATION_CAP
      ? scope
      : scope.filter(n => !isHighConfidence(n.output?.confidence))

  const uncovered = mustAddress.filter(n => !addressedBy(artifact, n.id))
  if (uncovered.length > 0) {
    return err(
      'uncovered_siblings',
      `Gate ${gateId} cannot pass: you never named ${uncovered.map(n => n.id).join(', ')}. ` +
      `An audit that does not mention what it audited is a rubber stamp. ` +
      `Address each by id, or inject a gap node.`
    )
  }

  return ok(undefined)
}

/** A gate passing = completing, but only after the three checks. */
export function passGate(graph, gateId, actor, artifact) {
  const valid = validateGatePass(graph, gateId, artifact)
  if (!valid.ok) return valid
  return completeNode(graph, gateId, actor, artifact)
}

/**
 * The alternative to passing: the gate found a hole and adds work.
 * New nodes carry origin 'gap', and whatever waited on the gate now waits on
 * the new work too.
 * jcode: ops.rs:444-540
 */
export function injectFromGate(graph, gateId, actor, specs) {
  const gate = graph.get(gateId)
  if (!gate) return err('unknown_node', `No such gate: ${gateId}`)
  if (!gate.isGate) return err('wrong_status', `Node ${gateId} is not a gate`)
  if (gate.owner !== null && gate.owner !== actor) {
    return err('not_owner', `Gate ${gateId} is owned by ${gate.owner}, not ${actor}`)
  }
  if (specs.length === 0) {
    return err('invalid_spec', 'Injecting requires at least one node')
  }

  const staged = graph.clone()
  const newIds = []

  for (const spec of specs) {
    if (staged.has(spec.id)) {
      return err('duplicate_id', `Node id already exists: ${spec.id}`)
    }
    staged.insert({
      id: spec.id,
      kind: spec.kind,
      origin: 'gap',                 // <- the growth signal
      status: 'queued',
      title: spec.title,
      scope: spec.scope,
      dependsOn: spec.dependsOn ? [...spec.dependsOn] : [],
      parent: gate.parent,
      isGate: false,
      owner: null,
      output: null,
    })
    newIds.push(spec.id)
  }

  // The gate re-runs once the gap work lands.
  staged.patch(gateId, {
    dependsOn: [...new Set([...gate.dependsOn, ...newIds])],
    status: 'queued',
    owner: null,
    output: null,
  })

  // Whoever waited on the gate now waits on the gap work too.
  if (gate.parent) {
    const parent = staged.get(gate.parent)
    if (parent) {
      staged.patch(gate.parent, {
        dependsOn: [...new Set([...parent.dependsOn, ...newIds])],
      })
    }
  }

  if (wouldCycle(staged)) {
    return err('cycle', 'Those gap nodes would create a cycle')
  }

  commitStaged(graph, staged)
  return ok(newIds)
}

/** Turn a completed node's admitted gaps into concrete node specs. */
export function gapSpecsFrom(node, prefix = 'gap') {
  return (node?.output?.whatIDidNotCheck ?? []).map((gap, i) => ({
    id: `${prefix}-${node.id}-${i + 1}`,
    kind: 'explore',
    title: gap.slice(0, 60),
    scope: `Cover what ${node.id} explicitly did not check: ${gap}`,
  }))
}
```

**Read the error strings again.** Each one names the unaddressed ids and offers
two legal ways forward: address them, or inject a gap. That is not politeness -
it is what makes the loop converge. A gate that returns "rejected" produces an
agent that resubmits the same artifact until `maxTurns`. A gate that returns
"you never named node-3, node-7" produces an agent that names node-3 and node-7.

**And notice what the machinery makes true.** A reviewer cannot pass by writing
"looks good". It cannot pass by listing everything as unchecked. It cannot pass
while its scope is still moving. The only ways forward are a real audit or more
work - and that property comes from the data model, not from prompt wording,
which is why it holds even when the model is careless.

---

# STEP 29 - `src/dag/scheduler.js`

**What jcode does:** `crates/jcode-plan/src/dag/schedule.rs` - `is_terminal` at
`:15`, `ready_nodes` at `:21`, `dispatch` at `:44`, `assemble_input` at `:64`.

**Three jobs:** decide what can run now, hand a node to a worker, and build that
worker's prompt from its dependencies' artifacts.

**`assembleInput` is the dataflow, and it is why this is a graph and not a
list.** Without it node B re-derives everything node A already learned.

```javascript
// src/dag/scheduler.js

export function isTerminal(node) {
  return node.status === 'done' || node.status === 'failed'
}

/** Queued, unowned, all dependencies done. "Blocked" is the absence of this. */
export function readyNodes(graph) {
  return graph.all().filter(node => {
    if (node.status !== 'queued') return false
    if (node.owner !== null) return false
    return node.dependsOn.every(id => graph.get(id)?.status === 'done')
  })
}

/** Claim a node for a worker. Returns false if someone else got there first. */
export function dispatch(graph, nodeId, worker) {
  const node = graph.get(nodeId)
  if (!node) return false
  if (node.status !== 'queued' || node.owner !== null) return false

  graph.patch(nodeId, { status: 'running', owner: worker })
  return true
}

/** Build a node's prompt from its own scope plus its dependencies' artifacts. */
export function assembleInput(graph, nodeId) {
  const node = graph.get(nodeId)
  if (!node) return ''

  const parts = [`# Task: ${node.title}`, '', node.scope, '']

  const upstream = node.dependsOn
    .map(id => graph.get(id))
    .filter(n => n && n.output)

  if (upstream.length > 0) {
    parts.push('## Results from the work you depend on', '')
    for (const dep of upstream) {
      parts.push(`### ${dep.id} - ${dep.title}`)
      parts.push(dep.output.findings)
      if (dep.output.confidence) parts.push(`confidence: ${dep.output.confidence}`)
      const gaps = dep.output.whatIDidNotCheck ?? []
      if (gaps.length > 0) parts.push(`did NOT check: ${gaps.join('; ')}`)
      parts.push('')
    }
  }

  if (node.isGate) {
    const audited = node.dependsOn.filter(id => !graph.get(id)?.isGate)
    parts.push(
      '## You are a gate',
      '',
      `You must address EVERY one of these by id: ${audited.join(', ')}`,
      '',
      'Pass only if you genuinely audited each one. If anything is uncovered,',
      'inject a gap node instead of passing. Listing something under',
      'whatIDidNotCheck does NOT count as addressing it.'
    )
  }

  return parts.join('\n')
}

/** Nothing ready and nothing running. */
export function isComplete(graph) {
  return readyNodes(graph).length === 0 &&
         graph.all().every(n => n.status !== 'running')
}
```

**`dispatch` returns a boolean rather than throwing** because two workers
polling `readyNodes` at the same moment both see the same node. The first
`dispatch` wins, the second gets `false` and moves on. That check-and-claim is
the only mutual exclusion this design needs, and it works precisely *because*
`dispatch` is synchronous - no `await` between reading `owner` and writing it,
so nothing can interleave. The Step 7 rule paying off.

**Checkpoint - Part 4 works.** These are the tests worth writing carefully:

```javascript
// test/dag.test.js
import { describe, it, expect } from 'vitest'
import { TaskGraph } from '../src/dag/TaskGraph.js'
import { seed } from '../src/dag/ops.js'
import { completeNode } from '../src/dag/complete.js'
import { passGate, mentionsNodeId } from '../src/dag/gates.js'

const artifact = (findings, confidence = 'high') => ({ findings, confidence })

describe('gates', () => {
  it('rejects a rubber stamp', () => {
    const graph = new TaskGraph('deep')
    seed(graph, [
      { id: 'a', kind: 'explore', title: 'A', scope: 'look at A' },
      { id: 'b', kind: 'explore', title: 'B', scope: 'look at B' },
    ])
    completeNode(graph, 'a', 'w1', artifact('Found the A subsystem, it uses REST.'))
    completeNode(graph, 'b', 'w2', artifact('Found the B subsystem, it uses gRPC.'))

    const result = passGate(graph, 'root::gate', 'reviewer', artifact('All good, no gaps found.'))

    expect(result.ok).toBe(false)
    expect(result.error.code).toBe('uncovered_siblings')
  })

  it('does not count whatIDidNotCheck as coverage', () => {
    // Gate artifact: findings 'Reviewed.', whatIDidNotCheck: ['a','b'].
    // Expect rejection - this is the loophole the rule exists to close.
  })

  it('will not pass over unaddressed low confidence', () => {
    // Complete 'a' with confidence 'low', gate names only 'b'.
    // Expect code 'unaddressed_low_confidence'.
  })
})

describe('mentionsNodeId', () => {
  it('matches whole tokens only', () => {
    expect(mentionsNodeId('checked node-a thoroughly', 'node-a')).toBe(true)
    expect(mentionsNodeId('checked node-ab thoroughly', 'node-a')).toBe(false)
    expect(mentionsNodeId('checked node-a.', 'node-a')).toBe(true)
  })
})
```

Write all of them, especially the `whatIDidNotCheck` one. If it passes when it
should fail, your gate has a loophole a careless model will find on its own.

---
---

# PART 5 - RUN IT AND WATCH IT

All the machinery exists. What is missing is the thing that makes a swarm feel
real: six agents working at once, and a gate rejecting a lazy audit in front of
you.

---

# STEP 30 - `src/services/swarm/SwarmRunner.js`

The piece nothing else covers: pull ready nodes off the graph, hand each to a
worker, feed the result back in.

**Mental model:** a worker pool over a queue that **grows while you drain it** -
because workers expand nodes and gates inject gaps. That is exactly why
`readyNodes` is recomputed every pass instead of captured once.

```javascript
// src/services/swarm/SwarmRunner.js
import { readyNodes, dispatch, assembleInput, isComplete } from '../../dag/scheduler.js'
import { completeNode, failNode } from '../../dag/complete.js'
import { passGate, injectFromGate, gapSpecsFrom } from '../../dag/gates.js'

/**
 * Parses a worker's final text into an artifact.
 *
 * Anything unparseable becomes a LOW-confidence artifact rather than an
 * exception, so a sloppy worker shows up as a debt the gates will chase
 * instead of as a crash. Failure stays visible.
 */
export function parseArtifact(text) {
  const wrapped = String(text ?? '').match(/<artifact>([\s\S]*?)<\/artifact>/)
  const raw = wrapped ? wrapped[1] : String(text ?? '')

  try {
    const parsed = JSON.parse(raw.trim())
    if (parsed && typeof parsed.findings === 'string') return parsed
  } catch {
    // fall through
  }

  return {
    findings: String(text ?? '').trim() || '(no findings reported)',
    confidence: 'low',
    whatIDidNotCheck: ['worker did not return a structured artifact'],
  }
}

/**
 * Drives the graph to completion using swarm members as workers.
 *
 * @param {import('../../dag/TaskGraph.js').TaskGraph} graph
 * @param {import('../../swarm/Swarm.js').Swarm} swarm
 * @param {string} coordinatorId
 * @param {{ maxParallel: number, maxPasses?: number, onDispatch?: Function, onResult?: Function }} options
 */
export async function runGraph(graph, swarm, coordinatorId, options) {
  const maxPasses = options.maxPasses ?? 100
  let passes = 0
  let completed = 0

  while (!isComplete(graph) && passes < maxPasses) {
    passes++

    const ready = readyNodes(graph).slice(0, options.maxParallel)
    if (ready.length === 0) break

    // Fan out: every worker starts before any is awaited.
    const inFlight = []
    for (const node of ready) {
      const spawned = swarm.spawn({
        requesterId: coordinatorId,
        prompt: assembleInput(graph, node.id),
        taskLabel: node.id,
      })
      if (!spawned.ok) continue

      const workerId = spawned.member.sessionId
      if (!dispatch(graph, node.id, workerId)) continue

      options.onDispatch?.(node.id, workerId)
      inFlight.push({ nodeId: node.id, workerId, isGate: node.isGate })
    }

    if (inFlight.length === 0) break

    // Fan in.
    const results = await Promise.all(
      inFlight.map(async task => ({ ...task, text: await swarm.join(task.workerId) }))
    )

    // Feed each result back into the graph.
    for (const task of results) {
      const artifact = parseArtifact(task.text)

      const outcome = task.isGate
        ? passGate(graph, task.nodeId, task.workerId, artifact)
        : completeNode(graph, task.nodeId, task.workerId, artifact)

      if (outcome.ok) {
        completed++
        options.onResult?.(task.nodeId, true, artifact.findings.slice(0, 80))

        // An honest admission becomes real work.
        const node = graph.get(task.nodeId)
        const gaps = gapSpecsFrom(node)
        if (gaps.length > 0 && node?.parent) {
          const gateId = `${node.parent}::gate`
          if (graph.has(gateId)) injectFromGate(graph, gateId, task.workerId, gaps)
        }
        continue
      }

      options.onResult?.(task.nodeId, false, outcome.error.message)

      if (task.isGate) {
        // A rejected gate re-runs; the next worker sees why in assembleInput.
        graph.patch(task.nodeId, { status: 'queued', owner: null })
      } else {
        failNode(graph, task.nodeId, task.workerId, outcome.error.message)
      }
    }
  }

  return { passes, completed }
}
```

**`graph.patch` is marked `@internal` and the runner calls it.** That is a real
seam: the runner is inside the engine's trust boundary, agents are not. If you
want that enforced rather than documented, move the re-queue into a named
`requeueGate` op in `gates.js` and keep `patch` genuinely private. Same lesson
as Step 25, one level up.

---

# STEP 31 - `src/components/SwarmView.jsx`

**What jcode does:** a live widget showing agents, status, and current task,
updating from event streams (docs/SWARM_ARCHITECTURE.md, "UI (TUI)").

**Why bother:** concurrency you cannot see is concurrency you cannot debug. A
log line saying "worker-3 completed" tells you nothing about whether four
agents ran together or one at a time.

```jsx
// src/components/SwarmView.jsx
import React, { useEffect, useState } from 'react'
import { Box, Text } from 'ink'
import { childrenOf, rootsOf } from '../swarm/ancestry.js'
import { SwarmEvents } from '../events/swarmEvents.js'

const COLOR = {
  spawned: 'gray', ready: 'gray', running: 'yellow', blocked: 'magenta',
  completed: 'green', failed: 'red', stopped: 'gray', crashed: 'red',
}

const MARK = {
  spawned: 'o', ready: 'o', running: '*', blocked: '!',
  completed: '+', failed: 'x', stopped: '-', crashed: 'X',
}

function MemberRow({ registry, member, depth }) {
  const kids = childrenOf(registry, member.sessionId)
  return (
    <Box flexDirection="column">
      <Text>
        {'  '.repeat(depth)}
        <Text color={COLOR[member.status]}>{MARK[member.status]} </Text>
        <Text bold>{member.friendlyName}</Text>
        <Text dimColor> {member.taskLabel ?? ''}</Text>
        <Text color={COLOR[member.status]}> [{member.status}]</Text>
      </Text>
      {kids.map(kid => (
        <MemberRow key={kid.sessionId} registry={registry} member={kid} depth={depth + 1} />
      ))}
    </Box>
  )
}

export function SwarmView({ swarm }) {
  const [, forceRender] = useState(0)

  useEffect(() => {
    const rerender = () => forceRender(n => n + 1)
    const unsub = swarm.bus.on(SwarmEvents.STATUS, rerender)
    const timer = setInterval(rerender, 250)   // catch changes that emit nothing
    return () => { unsub(); clearInterval(timer) }
  }, [swarm])

  const members = swarm.registry.all()
  const live = members.filter(m => m.status === 'running').length

  return (
    <Box flexDirection="column" borderStyle="round" paddingX={1}>
      <Text bold>Swarm</Text>
      <Text dimColor>{members.length} members, {live} running</Text>
      {rootsOf(swarm.registry).map(root => (
        <MemberRow key={root.sessionId} registry={swarm.registry} member={root} depth={0} />
      ))}
    </Box>
  )
}
```

**The interval alongside the subscription is not laziness.** Your `EventBus` is
synchronous and only fires on `swarm:status`, but a node being dispatched or a
gate being re-queued changes the DAG without touching member status. A slow
tick guarantees the view converges even where you forgot to emit. Correctness by
event, liveness by poll.

---

# STEP 32 - `src/components/DagView.jsx`

**The one thing this must show that a task list cannot:** the seeded/grown split
from Step 23. If a deep run ends with everything still `seed`, nothing
decomposed and no gate found anything - your rigor machinery never fired.

```jsx
// src/components/DagView.jsx
import React from 'react'
import { Box, Text } from 'ink'
import { readyNodes } from '../dag/scheduler.js'

const STATUS_COLOR = { queued: 'gray', running: 'yellow', done: 'green', failed: 'red' }
const ORIGIN_MARK  = { seed: 'S', expand: 'E', gap: 'G', gate: '#' }

export function DagView({ graph }) {
  const nodes = graph.all()
  const { seeded, grown } = graph.growthStats()
  const ready = readyNodes(graph).length
  const doneCount = nodes.filter(n => n.status === 'done').length

  return (
    <Box flexDirection="column" borderStyle="round" paddingX={1}>
      <Text bold>Task graph ({graph.mode} mode)</Text>
      <Text dimColor>
        {doneCount}/{nodes.length} done, {ready} ready | seeded {seeded}, grown {grown}
      </Text>
      {grown === 0 && nodes.length > 0 && (
        <Text color="red">nothing grew past the seed - rigor machinery never fired</Text>
      )}
      <Box marginTop={1} flexDirection="column">
        {nodes.map(node => (
          <Text key={node.id}>
            <Text dimColor>{ORIGIN_MARK[node.origin]} </Text>
            <Text color={STATUS_COLOR[node.status]}>{node.status.padEnd(7)}</Text>
            <Text bold>{node.id.padEnd(22)}</Text>
            <Text dimColor>
              {node.dependsOn.length > 0 ? `<- ${node.dependsOn.join(', ')}` : ''}
            </Text>
          </Text>
        ))}
      </Box>
      <Text dimColor>S=seed E=expand G=gap #=gate</Text>
    </Box>
  )
}
```

---

# STEP 33 - `src/screens/REPL.jsx`

Your existing REPL renders `MessageList`, `PromptInput`, and `StatusBar`. Add
the two swarm panels above the message list, gated on whether a swarm is
running so single-agent mode looks exactly as it does today:

```jsx
// src/screens/REPL.jsx - the additions
import { SwarmView } from '../components/SwarmView.jsx'
import { DagView } from '../components/DagView.jsx'

// ...inside the returned JSX, above <MessageList />:
{swarm && <SwarmView swarm={swarm} />}
{graph && <DagView graph={graph} />}
```

Keep `MessageList` bound to the **root** member's messages. Rendering every
agent's transcript inline is unreadable at four agents and actively harmful at
twenty - which is exactly why Step 21 built three separate read tiers instead
of one.

---

# STEP 34 - `src/entrypoints/cli.jsx` and the demo

**What the demo script deliberately does:** the first worker admits it did not
check something (so a gap node gets injected), and the first gate attempt is a
rubber stamp (so you watch it get rejected by name). Those two behaviours are
what separate this from a task list, so the demo should make them impossible to
miss.

```javascript
// src/demo/multimonitor.js

const artifact = (obj) => `<artifact>${JSON.stringify(obj, null, 2)}</artifact>`

/**
 * A worker script keyed off what the assembled prompt asks for.
 * Each member gets its own mock client, so `turn` is per-agent.
 */
export const multimonitorScript = (turn, params) => {
  const prompt = (params.messages ?? [])
    .map(m => (typeof m.content === 'string' ? m.content : ''))
    .join('\n')

  if (prompt.includes('You are a gate')) {
    const match = prompt.match(/address EVERY one of these by id: (.+)/)
    const ids = match ? match[1].split(',').map(s => s.trim()) : []

    // FIRST attempt: a rubber stamp. This gets rejected by name.
    if (turn === 0) {
      return { text: artifact({ findings: 'Reviewed the work. All good, no gaps found.', confidence: 'high' }) }
    }

    // SECOND attempt: an actual audit naming every node.
    return {
      text: artifact({
        findings:
          'Audited each node individually. ' +
          ids.map(id => `${id}: reviewed, findings consistent with its scope.`).join(' '),
        confidence: 'high',
      }),
    }
  }

  if (prompt.includes('display detection')) {
    return {
      text: artifact({
        findings: 'Display detection uses an EDID probe at startup. Hotplug is handled by a udev listener.',
        confidence: 'medium',
        whatIDidNotCheck: ['behaviour when a monitor is unplugged mid-render'],
      }),
    }
  }

  if (prompt.includes('window placement')) {
    return {
      text: artifact({
        findings: 'Window placement stores absolute coordinates, which break when the monitor layout changes.',
        confidence: 'high',
      }),
    }
  }

  return {
    text: artifact({
      findings: `Completed: ${prompt.slice(0, 100).replace(/\n/g, ' ')}`,
      confidence: 'high',
    }),
  }
}
```

```jsx
// src/entrypoints/cli.jsx
import React from 'react'
import { render } from 'ink'
import { Swarm } from '../swarm/Swarm.js'
import { EventBus } from '../events/EventBus.js'
import { TaskGraph } from '../dag/TaskGraph.js'
import { seed } from '../dag/ops.js'
import { runGraph } from '../services/swarm/SwarmRunner.js'
import { createMockClientFactory } from '../services/api/mockClient.js'
import { multimonitorScript } from '../demo/multimonitor.js'
import { createStore } from '../state/store.js'
import { getDefaultAppState } from '../state/AppStateStore.js'
import { allTools } from '../tools.js'
import { SwarmView } from '../components/SwarmView.jsx'
import { DagView } from '../components/DagView.jsx'

const useMock = process.argv.includes('--mock')
const dagMode = process.argv.includes('--light') ? 'light' : 'deep'

if (!useMock) {
  console.error('Real model not wired into the demo yet - run with --mock')
  process.exit(1)
}

const store = createStore(getDefaultAppState())
const bus = new EventBus()

const swarm = new Swarm({
  swarmId: 'demo',
  folderPath: process.cwd(),
  tools: allTools,
  getAppState: store.getState,
  setAppState: store.setState,
  bus,
  policy: { mode: 'deep', maxLiveWorkers: 6 },
  systemPromptFor: () =>
    'You are a worker in a swarm. Do the task described, then reply with ONLY an ' +
    '<artifact>...</artifact> block containing JSON with: findings, confidence, ' +
    'and optionally whatIDidNotCheck (an array of things you did not examine).',
  createStreamFactory: createMockClientFactory(multimonitorScript, { latencyMs: 120 }),
})

const graph = new TaskGraph(dagMode)

seed(graph, [
  { id: 'display-detection', kind: 'explore', title: 'Display detection',
    scope: 'How does display detection work today?' },
  { id: 'window-placement', kind: 'explore', title: 'Window placement',
    scope: 'How does window placement work today?' },
  { id: 'synthesis', kind: 'synthesize', title: 'Plan multimonitor support',
    scope: 'Combine the findings into a plan.',
    dependsOn: ['display-detection', 'window-placement'] },
])

const root = swarm.createRoot('Add multimonitor support')

function App() {
  return (
    <>
      <SwarmView swarm={swarm} />
      <DagView graph={graph} />
    </>
  )
}

const ink = render(<App />)

const result = await runGraph(graph, swarm, root.sessionId, {
  maxParallel: 4,
  onResult: (nodeId, ok, detail) => {
    if (!ok) console.log(`REJECTED ${nodeId}: ${detail}`)
  },
})

await new Promise(r => setTimeout(r, 300))
ink.unmount()

const { seeded, grown } = graph.growthStats()
console.log(
  `\n${result.passes} passes, ${result.completed} nodes completed. ` +
  `Seeded ${seeded}, grown ${grown}.`
)
```

Run it:

```bash
node src/entrypoints/cli.jsx --mock
```

**What you should see, and what each thing proves:**

1. **Two workers running side by side** on the first pass - `display-detection`
   and `window-placement` have no dependencies, so `readyNodes` returns both and
   they dispatch together. Parallelism (Parts 1-2).
2. **A `REJECTED root::gate` line naming both node ids.** The gate's first
   artifact was "All good, no gaps found" and the coverage check refused it.
   Step 28 working, and the most satisfying line in the run.
3. **The gate re-running and passing** once it names each node.
4. **A `G`-marked gap node appearing** from `display-detection`'s admission
   about unplugging a monitor mid-render. The graph grew because a worker was
   honest.
5. **`synthesis` running last**, because it depends on both explorations and,
   in deep mode, on the gate.
6. **`grown` greater than zero** in the summary.

Then run with `--light` and watch the difference: no gates, no rejection, no
gap injection. Same engine, same scheduler, same members. Only the rigor
changed - exactly the claim Step 23 made.

**Going back to the real model** costs nothing: your existing
`services/api/client.js` already satisfies the contract the mock imitates. Drop
`createStreamFactory: () => createApiStream` into the `Swarm` options and set
`ANTHROPIC_API_KEY`. **Start with `maxLiveWorkers: 2`** - a six-way fan-out of
real model calls gets expensive faster than you expect, and rate limits arrive
sooner than that.

Two things will differ from the mock, and both are worth seeing. Real models
sometimes wrap prose around the artifact block, which is why `parseArtifact`
degrades to low confidence rather than throwing. And real gates argue back - a
rejected gate will sometimes insist it did audit everything. Read the
transcript when that happens: your error message is the only thing steering it.

---

---
---

# PART 6 - FROM DEMO TO DAILY DRIVER

Parts 0-5 give you a swarm that runs a scripted demo. This part is what stands
between that and pointing it at a repository you care about.

Seven steps. None of them are about agents talking to each other - they are
about the boring things that decide whether a tool is usable: not destroying
your files, knowing what a run cost, and turning "here is my task" into a graph
without hand-writing the nodes.

Do Step 35 before you ever run this against real code.

---

# STEP 35 - `src/services/permissions/rules.js`

**The problem, stated plainly:** you have built an agent that runs shell
commands and writes files, and then made six copies of it. Every
`checkPermissions` in your codebase currently returns `{ granted: true }`.

**Why a swarm cannot use an interactive prompt.** For one agent, "ask the user
Y/N" works fine - that is what your `AskUserQuestionTool` does. For six agents
running concurrently, you get six modal prompts racing for one terminal, and a
human who becomes the bottleneck the parallelism was supposed to remove. jcode
solves this with rules evaluated per call plus modes that pre-authorize whole
classes of action (`docs/SAFETY_SYSTEM.md`). Rules scale; prompts do not.

**What to write:**

```javascript
// src/services/permissions/rules.js
import path from 'node:path'

/**
 * @typedef {'auto'|'plan'|'confined'} PermissionMode
 *   auto     - allow everything (use only against a throwaway clone)
 *   plan     - reads and searches only; every write or command is refused
 *   confined - writes allowed, but only inside folderPath; dangerous shell refused
 */

/** Commands that are never worth the risk of being wrong about. */
const DENIED_COMMAND_PATTERNS = [
  /\brm\s+-rf?\s+\//,           // rm -rf /
  /\bsudo\b/,
  /\bmkfs\b/,
  /\bdd\s+if=/,
  /:\(\)\{.*\};:/,              // fork bomb
  /\bcurl\b[^|]*\|\s*(ba)?sh/,  // curl | sh
  /\bgit\s+push\b.*--force/,
  /\bgit\s+reset\s+--hard\b/,
  /\bshutdown\b|\breboot\b/,
]

const WRITE_TOOLS = new Set(['write_file', 'edit_file', 'bash'])

/**
 * The single gate every tool calls. Pure function - no I/O, no prompts,
 * safe to evaluate six times concurrently.
 *
 * @returns {{ granted: true } | { granted: false, reason: string }}
 */
export function evaluatePermission(toolName, input, sessionContext) {
  const mode = sessionContext?.permissionMode ?? 'confined'
  const root = sessionContext?.folderPath

  if (mode === 'auto') return { granted: true }

  if (mode === 'plan' && WRITE_TOOLS.has(toolName)) {
    return {
      granted: false,
      reason:
        `Plan mode: '${toolName}' is not permitted. Investigate and report what ` +
        `you would change, but do not change it.`,
    }
  }

  if (toolName === 'bash') {
    const command = String(input?.command ?? '')
    const hit = DENIED_COMMAND_PATTERNS.find(p => p.test(command))
    if (hit) {
      return {
        granted: false,
        reason:
          `Refused: that command matches a destructive pattern (${hit}). ` +
          `If you genuinely need it, ask the user rather than running it.`,
      }
    }
  }

  // Path confinement. The highest-value rule here by some distance: it is what
  // stops an agent writing to ~/.ssh or outside the project it was pointed at.
  if (root && (toolName === 'write_file' || toolName === 'edit_file')) {
    const target = path.resolve(root, String(input?.path ?? ''))
    const rootResolved = path.resolve(root)
    if (target !== rootResolved && !target.startsWith(rootResolved + path.sep)) {
      return {
        granted: false,
        reason: `Refused: ${target} is outside the session folder (${rootResolved}).`,
      }
    }
  }

  return { granted: true }
}
```

**Now wire it in, which takes two edits.**

First, `checkPermissions` currently receives only `getAppState`/`setAppState` -
it cannot see `sessionContext`, so it cannot know the folder or the mode. Fix
that in `src/services/tools/StreamingToolExecutor.js`, inside `startTool`:

```javascript
      const permResult = await definition.checkPermissions(tool.block.input, {
        getAppState: this.getAppState,
        setAppState: this.setAppState,
        sessionContext: this.sessionContext,     // <- add
      })
```

Second, every tool that touches the world delegates to the gate. For
`BashTool`, `WriteFileTool`, and `EditFileTool`:

```javascript
  async checkPermissions(input, context) {
    return evaluatePermission('bash', input, context.sessionContext)
  },
```

Finally, put the mode on the session in `createSessionContext`:

```javascript
    permissionMode: options.permissionMode ?? 'confined',
```

**A note on how this pays off in a swarm.** Because the refusal is a *string the
model reads*, a worker denied a write in plan mode does not crash - it reports
what it would have changed. That means `--plan` gives you a genuinely useful
mode: fan out ten agents across a codebase, let them investigate, and get back
ten reports with zero risk. That is the safest and often most valuable thing
this whole system does.

---

# STEP 36 - `src/services/swarm/planTask.js`

**The gap this closes:** everything in Part 5 assumes a graph already exists.
`cli.jsx` hardcodes `seed(graph, [...])`. For your own work you need the front
door: *your task* goes in, *seed nodes* come out.

**Why the model does this rather than you:** decomposition is the judgement
call. Hand-writing nodes for every task is the thing you built an agent to
avoid.

**The design constraint that keeps it honest:** the planner is a single,
ordinary model call with no tools. It is not an agent. Keeping it dumb means it
cannot wander, cannot spend money, and fails visibly.

```javascript
// src/services/swarm/planTask.js

const PLANNER_PROMPT = `You decompose a software task into 2-5 independent
investigation nodes for a team of agents.

Rules:
- Nodes must be independently investigable. If two nodes would need to talk to
  each other, they are one node.
- Prefer investigation over action: "understand how X works", not "change X".
- The LAST node must be a synthesize node that depends on all the others.
- Use kind 'explore' for investigation and 'synthesize' for the final rollup.
- ids must be short, lowercase, hyphenated, and unique.

Reply with ONLY a JSON array, no prose:
[
  {"id":"...","kind":"explore","title":"...","scope":"a full paragraph telling
   the agent exactly what to find out","dependsOn":[]},
  {"id":"synthesis","kind":"synthesize","title":"...","scope":"...",
   "dependsOn":["...","..."]}
]`

/** Runs one tool-free turn and returns the assistant's text. */
async function askOnce(createStream, systemPrompt, userText) {
  const stream = createStream({
    messages: [{ type: 'user', content: userText }],
    tools: [],
    systemPrompt,
  })

  let text = ''
  for await (const event of stream) {
    if (event.type === 'assistant_message') {
      text += (event.message.content ?? [])
        .filter(b => b.type === 'text')
        .map(b => b.text)
        .join('')
    }
  }
  return text
}

/**
 * Turns a task string into seed NodeSpecs.
 * @returns {Promise<{ ok: true, specs: Array } | { ok: false, reason: string, raw: string }>}
 */
export async function planTask({ task, createStream, folderPath }) {
  const raw = await askOnce(
    createStream,
    PLANNER_PROMPT,
    `Repository: ${folderPath}\n\nTask: ${task}`
  )

  const match = raw.match(/\[[\s\S]*\]/)
  if (!match) return { ok: false, reason: 'Planner did not return a JSON array', raw }

  let specs
  try {
    specs = JSON.parse(match[0])
  } catch (error) {
    return { ok: false, reason: `Planner JSON did not parse: ${error.message}`, raw }
  }

  // Validate before it reaches the graph. seed() would reject a bad shape
  // anyway, but the error is far clearer here.
  const ids = new Set()
  for (const spec of specs) {
    if (!spec?.id || !spec?.kind || !spec?.title || !spec?.scope) {
      return { ok: false, reason: `Node missing required fields: ${JSON.stringify(spec)}`, raw }
    }
    if (ids.has(spec.id)) {
      return { ok: false, reason: `Duplicate node id: ${spec.id}`, raw }
    }
    ids.add(spec.id)
  }

  for (const spec of specs) {
    for (const dep of spec.dependsOn ?? []) {
      if (!ids.has(dep)) {
        return { ok: false, reason: `Node ${spec.id} depends on unknown ${dep}`, raw }
      }
    }
  }

  return { ok: true, specs }
}
```

**Keep the `raw` field on failure.** When a planner misbehaves you want to read
exactly what it said, not a sanitised error. That five-second feedback loop is
most of what makes prompt iteration tolerable.

---

# STEP 37 - `src/services/swarm/CostTracker.js`

**Why this is not optional:** Anthropic's published figure for their own
multi-agent research system is roughly **15x** the tokens of a single-agent
run, and a misbehaving subagent that spawns more subagents multiplies that
again. You will not develop good instincts about when fan-out is worth it
unless you can see the number.

Your `query.js` already emits everything needed - `assistantMessage.usage`
carries `inputTokens`, `outputTokens`, and `cacheReadInputTokens`, and
`Swarm.runMember` republishes every event on the bus.

```javascript
// src/services/swarm/CostTracker.js
import { SwarmEvents } from '../../events/swarmEvents.js'

/** USD per million tokens. Check current pricing before trusting totals. */
const PRICING = {
  'claude-opus-4-5':   { input: 15.00, output: 75.00, cacheRead: 1.50 },
  'claude-sonnet-4-6': { input:  3.00, output: 15.00, cacheRead: 0.30 },
  'claude-haiku-4-5':  { input:  0.80, output:  4.00, cacheRead: 0.08 },
}

export class CostTracker {
  constructor(bus, model = 'claude-sonnet-4-6') {
    this.model = model
    this.perMember = new Map()
    this.unsubscribe = bus.on(SwarmEvents.AGENT_EVENT, ({ sessionId, event }) => {
      const usage = event?.usage
      if (!usage) return

      const current = this.perMember.get(sessionId) ?? { input: 0, output: 0, cacheRead: 0 }
      current.input     += usage.inputTokens ?? 0
      current.output    += usage.outputTokens ?? 0
      current.cacheRead += usage.cacheReadInputTokens ?? 0
      this.perMember.set(sessionId, current)
    })
  }

  totals() {
    const sum = { input: 0, output: 0, cacheRead: 0 }
    for (const usage of this.perMember.values()) {
      sum.input     += usage.input
      sum.output    += usage.output
      sum.cacheRead += usage.cacheRead
    }
    const rates = PRICING[this.model] ?? PRICING['claude-sonnet-4-6']
    const usd =
      (sum.input     * rates.input     / 1e6) +
      (sum.output    * rates.output    / 1e6) +
      (sum.cacheRead * rates.cacheRead / 1e6)

    return { ...sum, usd, members: this.perMember.size }
  }

  report() {
    const t = this.totals()
    return (
      `${t.members} members | ` +
      `in ${t.input.toLocaleString()} / out ${t.output.toLocaleString()} ` +
      `(cache ${t.cacheRead.toLocaleString()}) | ~$${t.usd.toFixed(4)}`
    )
  }

  stop() {
    this.unsubscribe()
  }
}
```

**Run the same task twice - once with `maxLiveWorkers: 1` and once with 4 - and
compare.** That single experiment will teach you more about when to fan out
than any amount of reading, including this document.

---

# STEP 38 - `src/services/swarm/Timeline.js`

**The question no other view answers:** did the agents actually run *at the same
time*? The member tree shows you the current frame. A log shows you an ordered
list. Neither distinguishes true parallelism from fast sequential execution -
and that distinction is exactly what breaks when a tool blocks the event loop
(Step 3) or when `isReadOnly` is set wrong (Step 1).

```javascript
// src/services/swarm/Timeline.js
import { SwarmEvents } from '../../events/swarmEvents.js'
import { isTerminalStatus } from '../../swarm/types.js'

export class Timeline {
  constructor(bus, registry) {
    this.registry = registry
    this.spans = []
    this.startedAt = Date.now()

    this.unsubscribe = bus.on(SwarmEvents.STATUS, ({ sessionId, status }) => {
      if (status === 'running') {
        this.spans.push({ sessionId, start: Date.now(), end: null })
        return
      }
      if (isTerminalStatus(status)) {
        const open = [...this.spans].reverse().find(s => s.sessionId === sessionId && !s.end)
        if (open) open.end = Date.now()
      }
    })
  }

  /** ASCII gantt. Stacked bars = real parallelism; staircase = something serialises. */
  render(width = 60) {
    const now = Date.now()
    const total = Math.max(1, now - this.startedAt)

    const lines = this.spans.map(span => {
      const end = span.end ?? now
      const startCol = Math.floor(((span.start - this.startedAt) / total) * width)
      const length = Math.max(1, Math.floor(((end - span.start) / total) * width))
      const name = this.registry.get(span.sessionId)?.friendlyName ?? span.sessionId
      const label = this.registry.get(span.sessionId)?.taskLabel ?? ''

      return (
        name.padEnd(12).slice(0, 12) + ' ' +
        ' '.repeat(startCol) + '#'.repeat(length) +
        `  ${((end - span.start) / 1000).toFixed(1)}s ${label}`
      )
    })

    return [
      `timeline (${(total / 1000).toFixed(1)}s total, ${this.spans.length} runs)`,
      ...lines,
    ].join('\n')
  }

  /** Peak concurrency actually observed. The number that proves it. */
  peakConcurrency() {
    const points = []
    for (const span of this.spans) {
      points.push({ t: span.start, delta: 1 })
      points.push({ t: span.end ?? Date.now(), delta: -1 })
    }
    points.sort((a, b) => a.t - b.t || a.delta - b.delta)

    let live = 0
    let peak = 0
    for (const point of points) {
      live += point.delta
      peak = Math.max(peak, live)
    }
    return peak
  }

  stop() {
    this.unsubscribe()
  }
}
```

**`peakConcurrency()` is your regression test for parallelism.** Assert it is
greater than 1 in a test and you will catch the day someone reintroduces a
blocking call, which is otherwise nearly invisible.

---

# STEP 39 - `src/tools/ChannelTool/ChannelTool.js`

**Closing a real gap:** Step 20 gave you `joinChannel`, but nothing exposes it,
so agents can post to channels they can never join. This is the smallest step
here and it makes topic groups actually reachable.

```javascript
// src/tools/ChannelTool/ChannelTool.js
import { z } from 'zod'
import { buildTool } from '../../Tool.js'
import { joinChannel } from '../../services/comms/routing.js'

export const ChannelTool = buildTool({
  name: 'channel',
  description:
    'Join a topic channel so you receive messages posted to it. ' +
    'Use when several agents are working on the same area.',

  inputSchema: z.object({
    name: z.string().describe('Channel name, e.g. "parser"'),
  }),

  async call(input, context) {
    const { swarm, sessionId } = context.sessionContext ?? {}
    if (!swarm) throw new Error('Swarm is not enabled for this session.')

    joinChannel(swarm, sessionId, input.name)
    const members = swarm.channels?.get(input.name)?.size ?? 1
    return { data: `Joined #${input.name} (${members} member(s)).` }
  },

  async checkPermissions() { return { granted: true } },
  isConcurrencySafe: () => true,
  isReadOnly: () => true,
})
```

**A note worth carrying:** jcode's own design docs mark channels as
*discouraged* - the guidance is to prefer DMs and task-graph artifacts, because
a channel is a place for agents to have a conversation instead of doing work,
and conversations cost tokens without producing artifacts. Build it, use it
sparingly, and notice if your agents start chatting more than they report.

---

# STEP 40 - Enforcing single-writer

Time to make the rule from the top of this document structural instead of
advisory.

**What changes:** `Swarm` currently hands the same `tools` array to every
member. Replace it with a function of the member, so the orchestrator and the
workers get different capabilities.

In `src/swarm/Swarm.js`, change the option and the one place it is used:

```javascript
// SwarmOptions: replace `tools: Array` with
//   toolsFor: (member) => Array

    const engine = new QueryEngine({
      tools: this.options.toolsFor(member),        // <- was this.options.tools
      getAppState: this.options.getAppState,
      // ...unchanged
    })
```

Then define the split:

```javascript
// src/swarm/roles.js
import { ReadFileTool }  from '../tools/ReadFileTool/ReadFileTool.js'
import { GrepTool }      from '../tools/GrepTool/GrepTool.js'
import { BashTool }      from '../tools/BashTool/BashTool.js'
import { WriteFileTool } from '../tools/WriteFileTool/WriteFileTool.js'
import { EditFileTool }  from '../tools/EditFileTool/EditFileTool.js'
import { SpawnTool }     from '../tools/SpawnTool/SpawnTool.js'
import { MessageTool }   from '../tools/MessageTool/MessageTool.js'
import { ChannelTool }   from '../tools/ChannelTool/ChannelTool.js'

/** Workers investigate and report. They cannot change the repository. */
export const WORKER_TOOLS = [ReadFileTool, GrepTool, MessageTool, ChannelTool]

/** The orchestrator decides, edits, and fans out. */
export const ORCHESTRATOR_TOOLS = [
  ReadFileTool, GrepTool, BashTool, WriteFileTool, EditFileTool,
  SpawnTool, MessageTool, ChannelTool,
]

/**
 * Default policy: single-writer.
 * Swap for `() => ORCHESTRATOR_TOOLS` if you want workers that edit - just
 * read the warning at the top of this document first.
 */
export function toolsForMember(member) {
  return member.role === 'coordinator' ? ORCHESTRATOR_TOOLS : WORKER_TOOLS
}
```

**Notice what this buys you beyond safety.** A worker without `write_file`
cannot be *asked* to make a change, so its reports become genuinely
informational, and the orchestrator sees a consistent set of findings rather
than a set of half-applied edits. The architecture stops relying on the model
being disciplined.

**Notice also that `BashTool` is orchestrator-only here.** That is deliberate
and slightly aggressive - `bash` is a write tool in disguise (`> file`,
`git checkout`, `npm install`). If your workers genuinely need to run tests,
give them a narrower tool that only runs your test command rather than
arbitrary shell.

---

# STEP 41 - Run it for real

Everything assembled, against an actual repository.

**This replaces `src/entrypoints/cli.jsx` from Step 34.** You want one
entrypoint, not two - `--mock` runs Step 34's scripted demo, and without it the
same file plans your real task and runs it against a real repository. Delete the
Step 34 version once this works; it existed only so you could exercise the swarm
before Part 6 existed.

```javascript
// src/entrypoints/cli.jsx  (replaces the Step 34 version)
import React from 'react'
import { render } from 'ink'
import { Swarm } from '../swarm/Swarm.js'
import { EventBus } from '../events/EventBus.js'
import { TaskGraph } from '../dag/TaskGraph.js'
import { seed } from '../dag/ops.js'
import { runGraph } from '../services/swarm/SwarmRunner.js'
import { planTask } from '../services/swarm/planTask.js'
import { createApiStream } from '../services/api/client.js'
import { createMockClientFactory } from '../services/api/mockClient.js'
import { multimonitorScript } from '../demo/multimonitor.js'
import { CostTracker } from '../services/swarm/CostTracker.js'
import { Timeline } from '../services/swarm/Timeline.js'
import { toolsForMember } from '../swarm/roles.js'
import { createStore } from '../state/store.js'
import { getDefaultAppState } from '../state/AppStateStore.js'
import { SwarmView } from '../components/SwarmView.jsx'
import { DagView } from '../components/DagView.jsx'

const useMock = process.argv.includes('--mock')
const task = process.argv.slice(2).filter(a => !a.startsWith('--')).join(' ')
const folderPath = process.env.SWARM_FOLDER ?? process.cwd()
const permissionMode = process.argv.includes('--plan') ? 'plan' : 'confined'
const maxLiveWorkers = Number(process.env.SWARM_WORKERS ?? 3)

if (!task && !useMock) {
  console.error('usage: node src/entrypoints/cli.jsx "your task" [--plan] [--mock]')
  process.exit(1)
}

// The only difference between the demo and a real run is where turns come from.
const createStreamFactory = useMock
  ? createMockClientFactory(multimonitorScript, { latencyMs: 120 })
  : () => createApiStream

// 1. Get seed nodes. Scripted in mock mode, planned by the model otherwise.
let specs
if (useMock) {
  specs = [
    { id: 'display-detection', kind: 'explore', title: 'Display detection',
      scope: 'How does display detection work today?' },
    { id: 'window-placement', kind: 'explore', title: 'Window placement',
      scope: 'How does window placement work today?' },
    { id: 'synthesis', kind: 'synthesize', title: 'Plan multimonitor support',
      scope: 'Combine the findings into a plan.',
      dependsOn: ['display-detection', 'window-placement'] },
  ]
} else {
  const planned = await planTask({ task, createStream: createApiStream, folderPath })
  if (!planned.ok) {
    console.error('Planning failed:', planned.reason)
    console.error('Model said:\n', planned.raw)
    process.exit(1)
  }
  specs = planned.specs
}

console.log(`Plan: ${specs.map(s => s.id).join(', ')}\n`)

// 2. Build the graph and the swarm.
const bus = new EventBus()
const store = createStore(getDefaultAppState())
const graph = new TaskGraph('deep')

const seeded = seed(graph, specs)
if (!seeded.ok) {
  console.error('Seeding failed:', seeded.error.message)
  process.exit(1)
}

const swarm = new Swarm({
  swarmId: `run-${Date.now()}`,
  folderPath,
  toolsFor: toolsForMember,
  getAppState: store.getState,
  setAppState: store.setState,
  bus,
  policy: { mode: 'light', maxLiveWorkers },
  systemPromptFor: (member) =>
    member.role === 'coordinator'
      ? 'You orchestrate a team of agents working on a codebase.'
      : 'You are an investigator on a team. Do the task described using read and grep. ' +
        'Then reply with ONLY an <artifact>...</artifact> block containing JSON with: ' +
        'findings, confidence, and optionally whatIDidNotCheck.',
  createStreamFactory,
})

// permissionMode reaches tools via sessionContext (Step 35).
swarm.defaultPermissionMode = permissionMode

const cost = new CostTracker(bus)
const timeline = new Timeline(bus, swarm.registry)

const root = swarm.createRoot(task)
const ink = render(
  <>
    <SwarmView swarm={swarm} />
    <DagView graph={graph} />
  </>
)

const result = await runGraph(graph, swarm, root.sessionId, {
  maxParallel: maxLiveWorkers,
  onResult: (nodeId, okFlag, detail) => {
    if (!okFlag) console.log(`REJECTED ${nodeId}: ${detail}`)
  },
})

await new Promise(r => setTimeout(r, 250))
ink.unmount()

const { seeded: s, grown: g } = graph.growthStats()
console.log('\n' + timeline.render())
console.log(`\npeak concurrency: ${timeline.peakConcurrency()}`)
console.log(`nodes: ${result.completed} completed, seeded ${s}, grown ${g}`)
console.log(`cost: ${cost.report()}`)

// The actual output: every finished node's findings.
for (const node of graph.all()) {
  if (node.status === 'done' && !node.isGate) {
    console.log(`\n=== ${node.id} - ${node.title} ===\n${node.output.findings}`)
  }
}
```

One more line in `createSessionContext` so the mode propagates from the swarm:

```javascript
    permissionMode: options.permissionMode ?? 'confined',
```

and in `Swarm.#createMember`, pass it through:

```javascript
    const sessionContext = createSessionContext({
      folderPath: this.options.folderPath,
      parentSignal: spec.parentSignal,
      permissionMode: this.defaultPermissionMode,      // <- add
      swarm: this,
      bus: this.bus,
    })
```

**Your first real run should be this:**

```bash
export ANTHROPIC_API_KEY=...
export SWARM_FOLDER=/path/to/a/repo/you/do/not/mind/breaking
export SWARM_WORKERS=2

node src/entrypoints/cli.jsx "Explain how authentication works in this codebase" --plan
```

`--plan` means no writes at all - agents can only read, grep, and report. It is
the safest possible first contact with real code, and "map an unfamiliar
codebase" is genuinely one of the best things this architecture does. Expect
2-4 workers, a gate rejection or two, and a few cents.

**Then, when you trust it,** drop `--plan` and give it something small and
verifiable: *"add input validation to the signup handler and run the tests"*.
Watch the timeline. Read the rejected gates.

**What to expect, honestly.** The first three or four real runs will be
disappointing in specific, informative ways: a worker will return prose instead
of an artifact, a gate will argue that it did audit everything, the planner
will produce four nodes that are really one node. Every one of those is a
prompt problem, not an architecture problem, and fixing them is the actual work
of building agents. The machinery you built is what makes those failures
*visible* instead of silent.

**Checklist before you call it done:**

- `--plan` mode on a real repo produces useful findings and touches nothing
- `timeline.peakConcurrency()` is greater than 1
- at least one gate rejection appears and the re-run passes
- `graph.growthStats().grown` is greater than 0
- the cost line appears and the number is not a surprise
- a deliberately dangerous instruction ("delete everything in /") is refused by
  Step 35 rather than attempted

---

---

# STEP 42 - Give the graph to the agents

This is the step that closes the last real architectural gap between your MVP
and jcode. Do it once Step 41 runs.

**What you have after Step 41:** `SwarmRunner` reads the worker's text, parses
an artifact, and calls `completeNode` *on the worker's behalf*. The graph grows
only in one mechanical way - gaps derived from `whatIDidNotCheck`.

**What jcode does instead:** the agent mutates the graph itself, through tool
calls, with itself as the actor:

```
tool schema           crates/jcode-app-core/src/tool/communicate.rs:1959
  "task_graph", "expand_node", "complete_node", "inject_gap"

dispatch              communicate.rs:2629-2718

server handler        crates/jcode-app-core/src/server/comm_graph.rs
  dag::expand_node(&mut graph, &node_id, &req_session_id, specs)      :368
  dag::complete_node(&mut graph, &node_id, &req_session_id, artifact) :444
  dag::inject_from_gate(&mut graph, &gate_id, &req_session_id, specs) :511
```

Note `&req_session_id` in every call - **the calling agent is the actor.** That
is exactly the `actor` parameter your `expandNode` and `completeNode` already
take and that the runner currently fills in with the worker's id from the
outside.

**Why this matters, concretely.** Without it, a worker that discovers its task
is four tasks has no way to say so - it can only write prose about it and hope
the gate notices. With it, the worker calls `expand_node`, four children appear,
the scheduler picks them up on the next pass, and the plan adapts to what was
actually found. That is the difference between a plan decided up front and a
plan that responds to evidence.

**It changes nothing about the swarm.** Members are still durable peer sessions
with their own history, tools, and lifecycle. This is about who edits the plan,
not about what an agent is.

## 1. Put the graph on the session

`Swarm` needs to know about the graph so tools can reach it. In
`src/swarm/Swarm.js`, accept it as an option and pass it down:

```javascript
// SwarmOptions: add
//   graph: TaskGraph

// in #createMember, inside createSessionContext({ ... }):
      graph: this.options.graph,
```

And add the field in `src/services/session/createSessionContext.js`:

```javascript
    graph: options.graph,
```

## 2. The tool

```javascript
// src/tools/GraphTool/GraphTool.js
import { z } from 'zod'
import { buildTool } from '../../Tool.js'
import { expandNode } from '../../dag/ops.js'
import { completeNode } from '../../dag/complete.js'
import { injectFromGate, passGate } from '../../dag/gates.js'
import { assembleInput } from '../../dag/scheduler.js'

const nodeSpec = z.object({
  id: z.string(),
  kind: z.enum(['explore', 'implement', 'verify', 'fix', 'synthesize', 'critique']),
  title: z.string(),
  scope: z.string(),
  dependsOn: z.array(z.string()).optional(),
})

const artifact = z.object({
  findings: z.string(),
  confidence: z.string(),
  whatIDidNotCheck: z.array(z.string()).optional(),
  openQuestions: z.array(z.string()).optional(),
  validation: z.string().optional(),
})

export const GraphTool = buildTool({
  name: 'graph',
  description:
    'Read and change the task graph. Use expand_node when your task turns out to ' +
    'be several tasks, complete_node when you are finished, and inject_gap (gates ' +
    'only) when you find work that is missing.',

  inputSchema: z.object({
    action: z.enum(['task_graph', 'expand_node', 'complete_node', 'inject_gap']),
    node_id: z.string().optional(),
    specs: z.array(nodeSpec).optional(),
    artifact: artifact.optional(),
  }),

  async call(input, context) {
    const { graph, sessionId } = context.sessionContext ?? {}
    if (!graph) throw new Error('No task graph on this session.')

    // The actor is ALWAYS the calling session. An agent cannot claim to be
    // another agent, because this value never comes from tool input.
    const actor = sessionId

    switch (input.action) {
      case 'task_graph': {
        const lines = graph.all().map(n =>
          `${n.id} [${n.status}] ${n.isGate ? '(gate) ' : ''}${n.title}` +
          (n.dependsOn.length ? ` <- ${n.dependsOn.join(', ')}` : '')
        )
        const { seeded, grown } = graph.growthStats()
        return { data: `${lines.join('\n')}\n\nseeded ${seeded}, grown ${grown}` }
      }

      case 'expand_node': {
        if (!input.node_id || !input.specs?.length) {
          return { data: 'expand_node needs node_id and a non-empty specs array.' }
        }
        const result = expandNode(graph, input.node_id, actor, input.specs)
        return result.ok
          ? { data: `Expanded ${input.node_id} into: ${result.value.join(', ')}. ` +
                    `They will be scheduled once their dependencies are done.` }
          : { data: `Rejected (${result.error.code}): ${result.error.message}` }
      }

      case 'complete_node': {
        if (!input.node_id || !input.artifact) {
          return { data: 'complete_node needs node_id and an artifact.' }
        }
        const node = graph.get(input.node_id)
        // Gates go through the three checks; ordinary nodes do not.
        const result = node?.isGate
          ? passGate(graph, input.node_id, actor, input.artifact)
          : completeNode(graph, input.node_id, actor, input.artifact)

        return result.ok
          ? { data: `Completed ${input.node_id}.` }
          : { data: `Rejected (${result.error.code}): ${result.error.message}` }
      }

      case 'inject_gap': {
        if (!input.node_id || !input.specs?.length) {
          return { data: 'inject_gap needs node_id (the gate) and a specs array.' }
        }
        const result = injectFromGate(graph, input.node_id, actor, input.specs)
        return result.ok
          ? { data: `Injected: ${result.value.join(', ')}. This gate will re-run ` +
                    `after they finish.` }
          : { data: `Rejected (${result.error.code}): ${result.error.message}` }
      }

      default:
        return { data: `Unknown action: ${input.action}` }
    }
  },

  async checkPermissions() { return { granted: true } },

  // Graph mutations must not interleave: two expand_node calls staging from the
  // same clone would lose one of them.
  isConcurrencySafe: () => false,
  isReadOnly: () => false,
})
```

Add `GraphTool` to both `WORKER_TOOLS` and `ORCHESTRATOR_TOOLS` in
`src/swarm/roles.js`. It edits the *plan*, not the repository, so it does not
violate single-writer.

## 3. The runner stops writing results

`runGraph` currently parses the worker's text and writes the outcome itself.
Now the worker has done that, so the runner's job is only to **notice**.

Replace the result-handling block in
`src/services/swarm/SwarmRunner.js` with:

```javascript
    // Feed each result back into the graph.
    for (const task of results) {
      const node = graph.get(task.nodeId)

      // The worker completed it via the graph tool. Nothing to do.
      if (node?.status === 'done') {
        completed++
        options.onResult?.(task.nodeId, true, node.output?.findings?.slice(0, 80) ?? '')

        const gaps = gapSpecsFrom(node)
        if (gaps.length > 0 && node.parent) {
          const gateId = `${node.parent}::gate`
          if (graph.has(gateId)) injectFromGate(graph, gateId, task.workerId, gaps)
        }
        continue
      }

      // The worker expanded it instead of finishing it. Legitimate: the node
      // is queued again and now waits on its new children.
      if (node?.status === 'queued' && graph.childrenOf(task.nodeId).length > 0) {
        options.onResult?.(task.nodeId, true, 'expanded into children')
        continue
      }

      // The worker finished its turn without recording anything. Fall back to
      // parsing its text, so a model that ignores the tool still makes progress.
      const artifact = parseArtifact(task.text)
      const outcome = task.isGate
        ? passGate(graph, task.nodeId, task.workerId, artifact)
        : completeNode(graph, task.nodeId, task.workerId, artifact)

      if (outcome.ok) {
        completed++
        options.onResult?.(task.nodeId, true, artifact.findings.slice(0, 80))
        continue
      }

      options.onResult?.(task.nodeId, false, outcome.error.message)
      if (task.isGate) {
        graph.patch(task.nodeId, { status: 'queued', owner: null })
      } else {
        failNode(graph, task.nodeId, task.workerId, outcome.error.message)
      }
    }
```

**Keep the fallback.** Models forget to call tools, especially early in a
session. Without it, a worker that writes a perfect report in prose fails its
node for a formatting reason - and you will spend an hour thinking the graph is
broken when the prompt is.

## 4. Tell the workers

The system prompt in `cli.jsx` changes, because the contract changed:

```javascript
  systemPromptFor: (member) =>
    member.role === 'coordinator'
      ? 'You orchestrate a team of agents working on a codebase.'
      : 'You are an investigator on a team. Investigate using read and grep.\n' +
        'When you are done, call the graph tool with action "complete_node", ' +
        'your node id, and an artifact containing findings, confidence, and ' +
        'whatIDidNotCheck.\n' +
        'If the task turns out to be several separate investigations, call ' +
        '"expand_node" with 2-4 child specs instead of trying to do all of it.\n' +
        'If you are a gate, either complete_node with an audit naming every ' +
        'node by id, or inject_gap with the work that is missing.',
```

`assembleInput` (Step 29) already tells each worker its node id and, for gates,
exactly which ids must be addressed - so the worker has everything it needs to
call the tool correctly.

## What to watch for on the first run

- A worker calling `expand_node` and the DagView growing `E`-marked nodes
  mid-run. That is adaptive replanning, and it is the thing this step bought.
- `Rejected (not_owner)` - a worker trying to complete a node it was not
  dispatched. The `actor` check is working.
- `Rejected (uncovered_siblings)` - now coming back to a *gate agent* which can
  read it and retry, rather than being handled by runner code. Watch it argue,
  then comply.

**Now you match jcode's architecture:** durable peer sessions that talk to each
other, and a task graph the agents themselves reshape as they learn. What is
left after this is scope - worktrees, MCP, compaction, multi-provider,
persistence - not foundation.

---

---
---

# THE SAME ARCHITECTURE, TWO SETS OF NAMES

Read this table once you have finished building. Every row is one idea wearing
two costumes - the naming here, the naming in the TypeScript companion, and the
naming in real jcode. If you can look across a row and see one concept rather
than three implementations, the mental model has landed. It is also the
translation key for reading the jcode source itself.

| Idea | Plan A (`JCODE_MVP_LEARNING_PLAN.md`) | Plan B (this one) | Real jcode |
|---|---|---|---|
| The loop | `query.ts` -> `query()` | `src/query.js` -> `query()` | `Agent` turn loop |
| Agent identity | `Session` (`session.ts`) | `QueryEngine` + `sessionContext` | `Session` + `Agent` |
| Tool contract | `execute()` async generator | `call()` + `checkPermissions()` | `Tool` trait |
| Parallel gate | `isConcurrencySafe(input)` | `isReadOnly(input)` | `is_concurrency_safe` |
| Tool runner | `executeToolCalls()` | `StreamingToolExecutor` | `batch.rs` + `FuturesUnordered` |
| Model boundary | `Model.stream()` | `createApiStream(params)` | `jcode-provider-core` |
| Fake model | `createMockModel` | `createMockClient` | (none - this is ours) |
| Pub/sub | `SwarmBus` (new class) | `EventBus` (already existed) | `Bus` over `tokio::broadcast` |
| Member table | `SwarmRegistry` (`Map`) | `SwarmRegistry` (`Map`) | `Arc<RwLock<HashMap>>` |
| Parent pointer | `reportBackTo` | `reportBackTo` | `report_back_to_session_id` |
| Detached work | `Map<SessionId, Promise>` | `swarm.running` `Map` | `RuntimeTaskScope` (`JoinSet`) |
| Cancellation | `AbortController` chain | `AbortController` on `sessionContext` | `CancellationToken` + `InterruptSignal` |
| Batch fan-in | `planFanOut` | `planFanOut` | `try_join_all` |
| Streaming fan-in | `drainAsCompleted` | `drainAsCompleted` | `FuturesUnordered` |
| Event fan-in | `awaitMembers` | `awaitMembers` | `broadcast` + `select!` |
| Runner | `runner.ts` | `SwarmRunner.js` | scheduler + workers |
| Task graph | `TaskGraph` (`dag/`) | `TaskGraph` (`dag/`) | `jcode-plan/src/dag/` |
| Screens | `ui/*.tsx` | `components/*.jsx` + `screens/REPL.jsx` | `jcode-tui-*` |
| Entrypoint | `cli.ts` | `entrypoints/cli.jsx` | `src/bin/` |

**What genuinely differs between the two plans, beyond names:**

- Plan A builds the agent loop in eight steps. Plan B inherits a working one and
  spends five steps fixing what a swarm exposes - the `spawnSync` freeze, the
  missing `AbortController`, the emergent concurrency in the executor.
- Plan A's tools stream progress natively. Plan B keeps `call()` -> `{ data }`
  and adds streaming as an option, so your existing tools never changed.
- Plan A's types are compiler-checked. Plan B's are JSDoc, so the DAG's
  guarantees are enforced at runtime by the validated mutations rather than at
  build time. Which is precisely why Part 4's tests matter more here.
- **Part 6 exists only here.** Permissions, the task planner, cost, timeline,
  single-writer roles, and the real-repository entrypoint have no Plan A
  equivalent. That is the part that turns a demo into something you use.

**What did not differ at all:** the swarm and the DAG. `reportBackTo`,
`canSpawn`, the three fan-in strategies, `validateGatePass`, `readyNodes` -
identical logic either way, because that layer is architecture rather than
language.

That last point is the useful discovery, and it is why reading the TypeScript
companion afterwards is cheap: you will recognise every line of Parts 1-5
wearing different names, which is the clearest possible evidence that what you
learned was the architecture and not the vocabulary.

---

# THE 5-MINUTE MENTAL MODEL

Eight things. If you can explain them without notes, you own this.

```
1. MEMBER:   a swarm member is a QueryEngine plus reportBackTo. There is no
             Agent class. The tree is derived by walking parent pointers,
             never stored - which is why reparenting is cheap and cannot
             desync from a second copy of the truth.

2. SPAWN:    creating an agent returns immediately. The child's first turn is
             detached, but the promise is kept in a Map so it can still be
             awaited, joined, or cancelled. Parallelism comes from not waiting
             at spawn - and from choosing deliberately where to wait instead.

3. FAN-IN:   three strategies, and picking right is the skill.
               planFanOut        - fixed set, want all, nothing to do meanwhile
               drainAsCompleted  - fixed set, want to react as each lands
               awaitMembers      - set changes while waiting; needs a deadline;
                                   re-derive truth from the registry each wake

4. CANCEL:   AbortController rides on sessionContext and chains parent to
             child, so killing a subtree is one call. Shutdown means signal
             AND wait, with a grace period, then mark the stragglers crashed.

5. STATE:    a plain Map. Node's single thread removes the data race but not
             the stale read - state can change across any await, so re-derive
             rather than remember. dispatch() is safe precisely because it is
             synchronous.

6. COMMS:    EventBus carries events; routing decides reach. A broadcast
             defaults to your own subtree, not the whole swarm. Messages queue
             as soft interrupts delivered between turns. Completed agents do
             not wake for messages, or you have built a billable feedback loop.

7. DAG:      work is a graph with a private node list and a closed set of
             validated mutations: seed, expandNode, completeNode,
             injectFromGate. Nodes remember their origin, so seeded-vs-grown
             measures whether thinking actually happened.

8. GATES:    the reason for all of it. A reviewer node is auto-inserted,
             cannot be bypassed, and cannot pass unless it (a) has a settled
             scope, (b) addresses every low-confidence node by id, and (c)
             names every node it audited. Listing something as unchecked does
             not count as checking it. The rigor lives in the data model, not
             the prompt - which is why it holds when the model is careless.
```

---

# WHAT TO BUILD NEXT

**Wire the swarm into the REPL properly.** Right now the demo drives everything
from `cli.jsx`. Make `/swarm <task>` a REPL command so you can start a swarm
conversationally, watch it, and keep talking to the root while it runs.

**Real conflict detection.** You defined `SwarmEvents.FILE_TOUCH` and never
emit it. Emit it from `WriteFileTool`, and when two live members touch the same
path, DM them each other's ids and let them sort it out. jcode is deliberately
optimistic here: no locks, conflicts route to a conversation between the agents
involved.

**Expose the DAG as tools.** Today `SwarmRunner` drives the graph and agents
just do tasks. Give agents `expand_node`, `complete_node`, and `inject_gap` as
tools so they can restructure the plan mid-run - which is how jcode actually
works (`crates/jcode-app-core/src/tool/communicate.rs:2629-2718` dispatches
exactly these into the engine). This is the biggest single step toward the real
thing.

**Bring back compaction, per member.** You already have
`services/context/compactMessages.js` and it already works. Twenty agents with
long transcripts will blow your budget far faster than one will, so the
threshold that was comfortable for a single CLI is now wrong. Measure before
you tune it.

**Cost tracking across the swarm.** Sum usage per member and print a total when
the run ends. Nothing teaches the economics of fan-out faster than watching the
number.

**HITL in a swarm.** You kept `AskUserQuestionTool`. What happens when three
agents ask you something at once? Queue them, show who is asking, and let the
answer route back to the right member. This is a genuinely interesting design
problem that single-agent CLIs never pose.

**Persistence and resume.** `saveSnapshot` / `loadSnapshot` handle members;
extend to the graph and resume after a crash. `recoverStatus` already encodes
the hard-won rule: never restore a status that described a live process.

---

# FURTHER READING

- `JCODE_MVP_LEARNING_PLAN.md` - the same build in TypeScript with independent
  naming. Best read *after* this one, as a translation exercise.
- `SWARM_HANDBOOK.md` - the concepts behind both plans, with the Rust
  explained chapter by chapter.
- `docs/SWARM_ARCHITECTURE.md` - roles, lifecycle states, communication model.
- `docs/SWARM_TASK_GRAPH.md` - the DAG design. Section 9 is a worked example of
  a graph evolving; section 6 covers the rigor mechanics you built in Step 28.
- `crates/jcode-plan/src/dag/` - the real engine: `mod.rs` for types, `ops.rs`
  for mutations, `schedule.rs` for the scheduler. Under 1,700 lines total, and
  worth reading now that you have written your own.
- `crates/jcode-app-core/src/server/` - `comm_session.rs` (spawn),
  `comm_await.rs` (fan-in), `swarm.rs` (registry and lifecycle),
  `comm_sync.rs` (the three read tiers).

Citations in this document point at jcode as of commit `5ae238574`, and at
`data-swarm-cli` as it stood when this plan was written. Line numbers drift; if
one does not resolve, search for the function name instead - names have been far
more stable than positions.
