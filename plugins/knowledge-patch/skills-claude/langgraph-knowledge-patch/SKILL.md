---
name: langgraph-knowledge-patch
description: LangGraph
version: null
license: MIT
metadata:
  author: Nevaberry
---


# LangGraph

Use this skill when designing, migrating, debugging, or deploying LangGraph
graphs in Python or JavaScript/TypeScript. Start with the quick references for
high-impact compatibility changes, then open the topic reference that matches
the work.

## Reference index

| Reference | Topics |
| --- | --- |
| [references/migration-and-state.md](references/migration-and-state.md) | Agent migration, package requirements, state schemas, reducers, private and runtime-only state, runtime context |
| [references/execution-and-routing.md](references/execution-and-routing.md) | Routing, replay, retries, timeouts, error handlers, caching, recursion, migrations, execution metadata |
| [references/streaming-and-interrupts.md](references/streaming-and-interrupts.md) | Wire encoding, UI transports, v3 streams, privacy filtering, interrupt and resume patterns |
| [references/persistence-and-subgraphs.md](references/persistence-and-subgraphs.md) | Durability, checkpoint savers, serialization, delta channels, subgraph state and namespaces |
| [references/deployment.md](references/deployment.md) | Agent Server loading, storage roles, runtime layouts, queues, threadless runs, JavaScript deployment |

## Breaking changes and deprecations

### Build agents through LangChain

Replace the deprecated prebuilt React-agent factory with the LangChain agent
factory. The resulting agent still runs on LangGraph and supports middleware.

```python
from langchain.agents import create_agent

agent = create_agent(
    model,
    tools,
    system_prompt="You are a helpful assistant.",
)
```

```typescript
import { createAgent } from "langchain";

const agent = createAgent({
  model,
  tools,
  systemPrompt: "You are a helpful assistant.",
});
```

Use `system_prompt` in Python and `systemPrompt` in JavaScript instead of
`prompt`.

For Python migrations:

- Import `AgentState` from `langchain.agents`.
- Replace the Pydantic and structured-response state variants with
  `AgentState`.
- Rename `HumanInterruptConfig` and `ActionRequest` to `InterruptOnConfig`.
- Rename `HumanInterrupt` to `HITLRequest`.
- Remove `ValidationNode`; `create_agent` validates tool input automatically.
- Replace `MessageGraph` with a `StateGraph` whose state has a `messages` key.

JavaScript packages require Node.js 22 or newer. Python-side LangChain packages
require Python 3.10 or newer. JavaScript packages ship bundled output, so
replace private `dist/` imports with public module imports.

### Return encoded streams directly

The low-level `toLangGraphEventStream` helper is gone. Ask `graph.stream` for
the wire encoding and return that stream:

```typescript
const stream = await graph.stream(input, {
  encoding: "text/event-stream",
  streamMode: ["values", "messages"],
});

return new Response(stream, {
  headers: { "Content-Type": "text/event-stream" },
});
```

### Treat commands as additive routing

`Command(goto=...)` adds a destination; it does not cancel a node's static
outgoing edges. If both exist, both paths execute. Choose dynamic command
routing or static edges for a node, including nodes that process
tool-returned commands.

Any `Command` passed to `invoke` or `stream` resumes the latest checkpoint.
Use `Command(resume=...)`, optionally with `update`, only for a resume. Start a
new turn on an existing thread by passing a plain state mapping:

```python
graph.invoke({"messages": [follow_up]}, config)
graph.invoke(Command(resume=review_answer), config)
```

### Design replay-safe nodes

An interrupt or retry restarts the affected node from its beginning. Make
side effects before the restart point idempotent. Tasks within the node are
checkpointed and completed task results may be reused, but reordering tasks or
interrupts before a resume point can mismatch saved results.

## State and schema quick reference

### Prefer `StateSchema` in JavaScript

`StateSchema` accepts standard field schemas and LangGraph value types.
`MessagesValue` supplies message-aware reduction; `ReducedValue` combines a
field schema and default with a custom reducer.

```typescript
import { MessagesValue, ReducedValue, StateSchema } from "@langchain/langgraph";
import * as z from "zod";

const State = new StateSchema({
  messages: MessagesValue,
  total: new ReducedValue(z.number().default(0), {
    reducer: (current, update) => current + update,
  }),
});
```

Treat `Annotation.Root` and direct Zod v3/v4 integrations as legacy
alternatives.

### Keep runtime context outside state

In Python, declare `context_schema`, type `Runtime.context`, and pass
`context=` when invoking. In JavaScript, supply a context schema to the
`StateGraph` constructor, read `runtime.context`, and pass
`{ context: ... }` in invocation options.

Do not assume an input or output schema hides state in streams. A node input
schema restricts reads, not writes; node schemas can extend the state-channel
union. `values` snapshots include input, output, and private channels. For v3
events, select safe snapshot fields with `output_keys` or `outputKeys`.

Use Python `Overwrite(value)` to bypass a configured reducer for one update.
Use JavaScript `UntrackedValue` for execution-only objects that must not enter
checkpoints. Its default `guard: true` rejects multiple writes in one step;
`guard: false` allows them and keeps the last value.

## Reliability and node policy quick reference

### Configure both halves of caching

Caching needs a node cache policy and a cache on the compiled graph. Without
either, the node is not cached. An omitted TTL means no expiry. Python's
default key hashes the pickled node input; JavaScript configures `cachePolicy`
and `keyFunc` and imports `InMemoryCache` from
`@langchain/langgraph-checkpoint`.

### Select retries deliberately

Python's default retry filter excludes `ValueError`, `TypeError`,
`RuntimeError`, and `OSError`, and retries HTTP-library errors only for 5xx
responses. JavaScript excludes `TypeError`, `SyntaxError`, and
`ReferenceError`. Use `retry_on` or `retryOn` when those defaults do not match
the failure model.

Python async nodes can use per-attempt `timeout=` values. A timeout raises
`NodeTimeoutError`, discards buffered writes and child-task scheduling, and
may be retried with a fresh timer. A timeout on a synchronous node makes graph
compilation fail.

An `error_handler` runs only after a Python node fails and exhausts retries.
It receives state and a typed `NodeError`, and may return a `Command` that
updates state and routes to recovery.

Use `StateGraph.set_node_defaults()` for graph-wide Python defaults. Explicit
node settings win, and defaults do not propagate into subgraphs. Retry and
timeout defaults apply to handler nodes; cache and error-handler defaults
apply only to regular nodes.

## Interrupt quick reference

### Use one interrupt per node invocation

Do not place `interrupt()` in a validation `while` loop. Each resume restarts
the node and replays earlier loop iterations, so work grows rapidly. Store the
next prompt in state, call `interrupt()` once, and route back with a
conditional edge after invalid input.

When parallel branches pause, resume all pending interrupts by mapping each
interrupt `id` to its response. This keeps answers paired with the correct
branch.

For Python v3 streams, inspect `stream.interrupted` and `stream.interrupts`
after driving the stream to completion. Resume with a new
`stream_events(Command(resume=...), version="v3")` call and repeat until the
stream finishes without interruption.

## Persistence and subgraph quick reference

Choose durability per run:

- `exit` writes when execution completes, errors, or interrupts.
- `async` writes while the next step runs and can lose the latest checkpoint
  in a crash.
- `sync` persists before advancing.

Subgraph compilation controls state lifetime:

- Default `checkpointer=None` starts fresh for each call but inherits the
  parent checkpointer for interrupts and durable execution during that call.
- `checkpointer=True` retains state across calls on the same thread.
- `checkpointer=False` disables checkpointing, interrupts, durable execution,
  and state inspection.

The parent needs a checkpointer for either stateful mode. Never run two
concurrent calls to the same `checkpointer=True` subgraph: they share a
checkpoint namespace and conflict. Serialize access or use per-invocation
persistence.

## Deployment quick reference

Exported compiled graphs are loaded once at Agent Server startup and reused.
Graph factories run for every execution and are appropriate only when
per-run customization is required. The server injects its checkpointer and
memory store, so deployed graph code must not configure either one.

PostgreSQL always stores assistants, threads, runs, and cron jobs. Checkpoints
default to PostgreSQL but may use MongoDB or a custom backend. The long-term
Store also defaults to PostgreSQL and is replaceable. Redis is only for
ephemeral signaling, cancellation, and streaming pub/sub.

In a queue-backed deployment, the durable database leases runs to workers and
only one run per thread executes at a time. `N_JOBS_PER_WORKER` controls jobs
per worker and defaults to `10`; it does not limit API request concurrency.
Split deployments must keep at least one queue worker listening.
