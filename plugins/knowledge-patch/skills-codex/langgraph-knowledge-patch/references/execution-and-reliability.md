# Execution and Reliability

Source batches: `graph-api-overview`, `graph-api-usage`.

## Node caching

Caching activates only when the node has a cache policy and the compiled graph
has a cache. A policy without TTL never expires. Python's default cache key
hashes the pickled node input.

```python
from langgraph.cache.memory import InMemoryCache
from langgraph.types import CachePolicy

builder.add_node("expensive", expensive, cache_policy=CachePolicy(ttl=30))
graph = builder.compile(cache=InMemoryCache())
```

JavaScript uses `cachePolicy` and `keyFunc`. Import its `InMemoryCache` from
`@langchain/langgraph-checkpoint`.

## Replay and task ordering

After an interrupt or retry, LangGraph starts the affected node again from its
beginning. Make side effects before an interrupt idempotent.

Tasks within a node are checkpointed, so a completed task result can be reused
during replay. Do not reorder tasks or interrupts that precede a resume point:
saved results are positional enough that a changed order can mismatch them.

## Routing and invocation input

A node's `Command(goto=...)` adds a dynamic route and does not suppress
static outgoing edges. When both are configured, both destinations execute. Use
either `Command` routing or static edges for that node. The same additive rule
applies to commands returned by tools.

Passing any `Command` to `invoke` or `stream` resumes execution from the latest
checkpoint. `Command(resume=...)`, optionally with `update`, is the command
intended for invocation input. To begin a new turn on an existing thread from
`__start__`, pass a plain state mapping rather than `Command(update=...)`.

```python
graph.invoke({"messages": [follow_up]}, config)
graph.invoke(Command(resume=review_answer), config)
```

## Recursion budgets

Since LangGraph Python 1.0.6, the default recursion limit is 1000 super-steps.
JavaScript defaults to 25. Set `recursion_limit` in Python or
`recursionLimit` in JavaScript as a top-level invocation config field, not
inside `configurable`.

Nodes can inspect the current super-step through
`metadata.langgraph_step`. Python graphs can also declare a `RemainingSteps`
managed state field to route proactively before the budget is exhausted.

```python
result = graph.invoke(inputs, config={"recursion_limit": 100})
current_step = config["metadata"]["langgraph_step"]
```

## Retry filtering

Attach `RetryPolicy` with `retry_policy=` in Python or `retryPolicy` in
JavaScript.

Python's default filter excludes common programming and runtime exceptions,
including `ValueError`, `TypeError`, `RuntimeError`, and `OSError`. HTTP
library failures are retried only for 5xx responses. JavaScript's default
excludes `TypeError`, `SyntaxError`, and `ReferenceError`.

Use `retry_on` or `retryOn` when retries should deliberately include or
exclude different failures.

## Python node timeouts

With `langgraph>=1.2`, an asynchronous node's `timeout=` may be seconds, a
`timedelta`, or `TimeoutPolicy(run_timeout=..., idle_timeout=...)`.

```python
from langgraph.types import TimeoutPolicy

builder.add_node(
    "call_model",
    call_model,
    timeout=TimeoutPolicy(run_timeout=120, idle_timeout=30),
)
```

Each attempt receives a fresh timer. Timeout raises `NodeTimeoutError` and
discards buffered writes and child-task scheduling; a retry can then start a
new attempt. Configuring a timeout on a synchronous node fails graph
compilation.

## Post-retry error handling

With `langgraph>=1.2`, Python `add_node(error_handler=...)` installs a handler
that runs only when a node failure has exhausted its retries. The handler
receives current state and a typed `NodeError`. It may return a `Command` that
updates state and routes to compensation or recovery.

```python
from langgraph.errors import NodeError
from langgraph.types import Command

def recover(state: State, error: NodeError) -> Command:
    return Command(update={"status": str(error.error)}, goto="fallback")

builder.add_node("charge", charge, error_handler=recover)
```

## Graph-wide policy defaults

Python `langgraph>=1.2` provides `StateGraph.set_node_defaults()` for
compile-time `retry_policy`, `timeout`, `cache_policy`, and `error_handler`
defaults. An explicit per-node value wins, and defaults do not propagate into
subgraphs.

Retry and timeout defaults apply to handler nodes. Cache and error-handler
defaults apply only to regular nodes.

```python
builder.set_node_defaults(
    retry_policy=RetryPolicy(max_attempts=3),
    timeout=TimeoutPolicy(run_timeout=30),
    error_handler=recover,
)
```

## Execution and server metadata

Nodes can read execution identity and retry state through Python
`runtime.execution_info` or JavaScript `runtime.executionInfo`. The object can
include thread, run, checkpoint, task, attempt number, and first-attempt time.

Server deployments also provide assistant, graph, and authenticated-user data
through `server_info` or `serverInfo`. That server metadata is absent in local
execution. These fields require Python `langgraph>=1.1.5` or JavaScript
`@langchain/langgraph>=1.2.8`.

## Graceful drain

Python `langgraph>=1.2` exposes `runtime.drain_requested` and
`runtime.drain_reason` once a run drain has been requested. A node may inspect
them to skip expensive work before the next super-step boundary.

```python
def node(state: State, runtime: Runtime):
    if runtime.drain_requested:
        return {"status": "skipped", "reason": runtime.drain_reason}
    return {"status": do_work()}
```

## Deferred fan-in

In Python, `add_node(..., defer=True)` postpones the node until every pending
task has finished. Use it for a final fan-in when parallel branches have
different lengths; without it, a node can run as soon as one incoming branch
arrives.

```python
builder.add_node("finalize", finalize, defer=True)
```
