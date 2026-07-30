---
name: langgraph-knowledge-patch
description: LangGraph
version: null
license: MIT
metadata:
  author: Nevaberry
---


# LangGraph Knowledge Patch

Use this skill when maintaining LangGraph applications, especially for agent
migrations, graph execution semantics, durable state, human-in-the-loop flows,
subgraphs, streaming, or Agent Server deployment.

## Reference index

| Reference | Topics |
| --- | --- |
| [migrations-and-state.md](references/migrations-and-state.md) | Agent and prebuilt migrations, runtime requirements, schemas, context, state updates |
| [execution-and-reliability.md](references/execution-and-reliability.md) | Routing, replay, caching, recursion, retries, timeouts, handlers, deferred nodes |
| [streaming-and-interrupts.md](references/streaming-and-interrupts.md) | Wire streams, transports, private values, typed interrupts, resume patterns |
| [persistence-and-subgraphs.md](references/persistence-and-subgraphs.md) | Durability, custom savers, serialization, delta channels, subgraph lifetimes |
| [deployment.md](references/deployment.md) | Server loading, storage roles, runtime layouts, queues, remote streaming |

## Breaking changes and deprecations

### Build agents through LangChain

Use the LangChain agent factory. The old LangGraph React-agent prebuilt is
deprecated, although the replacement still executes on LangGraph.

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

Rename Python `prompt` to `system_prompt`; in JavaScript use `systemPrompt`.
Import `AgentState` from `langchain.agents`. Replace:

- Pydantic and structured-response state variants with `AgentState`.
- `HumanInterruptConfig` and `ActionRequest` with `InterruptOnConfig`.
- `HumanInterrupt` with `HITLRequest`.
- `ValidationNode` with `create_agent` tool-input validation.
- `MessageGraph` with `StateGraph` containing a `messages` key.

### Respect current runtime and package boundaries

- JavaScript packages require Node.js 22 or newer.
- Python-side LangChain packages require Python 3.10 or newer.
- JavaScript packages ship bundled output. Replace private `dist/` imports with
  public module imports.
- Prefer JavaScript `StateSchema`; treat `Annotation.Root` and direct Zod
  integrations as legacy alternatives.

## Execution semantics that prevent subtle bugs

### Do not mix static and dynamic routing

`Command(goto=...)` adds a destination; it does not cancel a node's static
outgoing edges. If both exist, both paths run. Choose static edges or
`Command` routing for a node, including nodes that return tool commands.

### Distinguish a new turn from a resume

Any `Command` passed to `invoke` or `stream` resumes the latest checkpoint.
Use `Command(resume=..., update=...)` for a resume. To begin a new turn on an
existing thread at `__start__`, pass a plain state mapping:

```python
graph.invoke({"messages": [follow_up]}, config)
graph.invoke(Command(resume=review_answer), config)
```

### Make replayed work safe

An interrupted or retried node starts again from its beginning. Make side
effects before the pause idempotent. Completed tasks within the node can be
reused from checkpoints, but changing task or interrupt order before a resume
point can associate saved results with the wrong operation.

### Configure recursion at the invocation top level

Python's default is 1000 super-steps from 1.0.6; JavaScript's default is 25.
Set `recursion_limit` or `recursionLimit` beside other invocation options, not
inside `configurable`. Read `metadata.langgraph_step` in a node, or declare
Python `RemainingSteps`, to route before exhausting the budget.

```python
result = graph.invoke(inputs, config={"recursion_limit": 100})
step = config["metadata"]["langgraph_step"]
```

## State, context, and emitted values

An input schema limits what a node reads, not which graph channels it may
update. Node schemas can also add private channels to the graph-state union.
Input, output, and private schemas do not redact `values` streams; for
sensitive snapshots, use v3 events and `output_keys` or `outputKeys`.

Use Python `Overwrite` to bypass a channel reducer for one update:

```python
from langgraph.types import Overwrite

return {"items": Overwrite(["replacement"])}
```

Keep immutable configuration and dependencies out of graph state. Declare
Python `context_schema` and read `Runtime.context`; in JavaScript supply the
context schema to `StateGraph` and read `runtime.context`.

Pydantic graph state validates only the input to the first node. Later updates
and graph output are not automatically validated, and `invoke` returns a
dictionary. Use `AnyMessage` for message fields that cross the wire.

JavaScript `UntrackedValue` holds runtime-only data and is absent from
checkpoints. Its default `guard: true` rejects multiple writes in one step;
`guard: false` allows them and retains the last write.

## Reliability policies

Caching needs both a node cache policy and a cache on the compiled graph.
Omitting TTL means no expiry. Python's default key hashes pickled node input;
JavaScript uses `cachePolicy` and `keyFunc`, with `InMemoryCache` from
`@langchain/langgraph-checkpoint`.

Default retry filtering intentionally excludes common programming failures.
Python excludes `ValueError`, `TypeError`, `RuntimeError`, and `OSError`, and
retries HTTP errors only for 5xx responses. JavaScript excludes `TypeError`,
`SyntaxError`, and `ReferenceError`. Set `retry_on` or `retryOn` explicitly
when those defaults are unsuitable.

Python asynchronous nodes can use per-attempt `timeout=` with a number of
seconds, `timedelta`, or `TimeoutPolicy`. A timeout raises `NodeTimeoutError`,
discards buffered writes and child scheduling, and may be retried with a fresh
timer. A timeout on a synchronous node is a compile error.

An `error_handler` runs only after node retries are exhausted. It receives the
current state and a typed `NodeError`, and may return a `Command` for recovery.
Use `StateGraph.set_node_defaults()` for graph-wide retry, timeout, cache, and
handler policies; explicit node settings win, and defaults do not enter
subgraphs.

## Interrupt and stream checklist

- Low-level JavaScript HTTP handlers should request
  `encoding: "text/event-stream"` from `graph.stream` and return it directly;
  `toLangGraphEventStream` has been removed.
- React `useStream` accepts a custom `transport`, such as
  `FetchStreamTransport`.
- JavaScript named interrupt definitions type both their request and resume
  values; use the compiled graph's interruption check.
- With parallel interrupts, resume with one mapping from every interrupt ID to
  its response.
- Call `interrupt()` at most once per node invocation. For validation, store
  the next prompt in state and use a conditional edge to revisit the node.
- In Python v3 event streams, inspect `interrupted` and `interrupts`, then
  start another stream with `Command(resume=...)` until it completes.

## Persistence and subgraph checklist

- Select `durability="exit"`, `"async"`, or `"sync"` based on the acceptable
  write timing and crash window.
- Custom checkpoint savers must implement exact-ID and latest reads, newest-
  first history, complete deletes, and serialization of checkpoints, writes,
  and metadata.
- Use pickle fallback only for unsupported serializer values. Use an encrypted
  serializer when checkpoints require encryption.
- Keep PostgreSQL saver `thread_id` values under 255 characters.
- A default-compiled subgraph has per-call state while inheriting the parent
  checkpointer; `checkpointer=True` persists per thread, and `False` disables
  checkpoint-dependent capabilities.
- Never run the same per-thread subgraph concurrently. Give persistent child
  calls stable, name-based namespaces.

## Deployment checklist

- Exported graphs load once at server startup; graph factories run for every
  run and are appropriate only for per-run customization.
- Let Agent Server inject its checkpointer and Store.
- PostgreSQL persists assistants, threads, runs, and cron jobs. Redis provides
  ephemeral signaling, cancellation, and streaming pub/sub.
- In split mode, set `queue.enabled: true` and keep at least one queue worker.
- A queue executes at most one run per thread. `N_JOBS_PER_WORKER` defaults to
  10 and limits jobs per worker, not API request concurrency.
- For a threadless remote stream, pass `None` as the thread identifier and the
  deployed graph name as the next argument.

Consult the topic references before changing production persistence,
interrupt, subgraph, or deployment behavior; they contain the complete
contracts and edge cases behind these quick checks.
