# Execution and routing

## Checkpoint replay and side effects

After an interrupt or retry, the affected node restarts from the beginning.
Any earlier side effect must therefore be idempotent.

Tasks inside a node are checkpointed, and the runtime can reuse results from
completed tasks. Keep task and interrupt order stable before a resume point;
changing their order can associate saved results with the wrong operation.

## Dynamic and static routing

A node's `Command(goto=...)` adds a dynamic route without suppressing static
outgoing edges. If the node has both, both destinations run. Use one routing
style for a node. The same additive rule applies when a tool returns a
command.

Any `Command` supplied to `invoke` or `stream` resumes from the latest
checkpoint. `Command(resume=...)`, optionally with `update`, is the command
intended as resume input. To begin a new turn on an existing thread from
`__start__`, pass a plain state mapping rather than `Command(update=...)`.

```python
graph.invoke({"messages": [follow_up]}, config)
graph.invoke(Command(resume=review_answer), config)
```

## Node caches

Caching is active only when both conditions hold:

1. The node has a cache policy.
2. The compiled graph has a cache.

An omitted TTL means the entry does not expire. Python's default key function
hashes the pickled node input.

```python
from langgraph.cache.memory import InMemoryCache
from langgraph.types import CachePolicy

builder.add_node("expensive", expensive, cache_policy=CachePolicy(ttl=30))
graph = builder.compile(cache=InMemoryCache())
```

JavaScript uses `cachePolicy` and `keyFunc`. Import its `InMemoryCache` from
`@langchain/langgraph-checkpoint`.

## Retry behavior

Attach `RetryPolicy` with `retry_policy=` in Python or `retryPolicy` in
JavaScript.

Python's default retry selection excludes common programming and runtime
failures, including `ValueError`, `TypeError`, `RuntimeError`, and `OSError`.
For HTTP-library failures it retries only 5xx responses. JavaScript excludes
`TypeError`, `SyntaxError`, and `ReferenceError`.

Use `retry_on` in Python or `retryOn` in JavaScript to deliberately select
other exceptions.

## Python node timeouts and recovery

### Per-attempt async timeout

With `langgraph>=1.2`, an async node's `timeout=` accepts seconds, a
`timedelta`, or `TimeoutPolicy(run_timeout=..., idle_timeout=...)`:

```python
from langgraph.types import TimeoutPolicy

builder.add_node(
    "call_model",
    call_model,
    timeout=TimeoutPolicy(run_timeout=120, idle_timeout=30),
)
```

A timeout raises `NodeTimeoutError`. It discards buffered writes and scheduled
child tasks and can be retried with a new timer. Applying a timeout to a
synchronous node fails graph compilation.

### Post-retry error handlers

With `langgraph>=1.2`, `add_node(error_handler=...)` installs a handler that
runs only after the node fails and exhausts retries. It receives the current
state and a typed `NodeError`; it may return a `Command` to update state and
route to recovery:

```python
from langgraph.errors import NodeError
from langgraph.types import Command

def recover(state: State, error: NodeError) -> Command:
    return Command(update={"status": str(error.error)}, goto="fallback")

builder.add_node("charge", charge, error_handler=recover)
```

### Graph-wide policy defaults

With `langgraph>=1.2`, use `StateGraph.set_node_defaults()` to set compile-time
defaults for `retry_policy`, `timeout`, `cache_policy`, and `error_handler`.

```python
builder.set_node_defaults(
    retry_policy=RetryPolicy(max_attempts=3),
    timeout=TimeoutPolicy(run_timeout=30),
    error_handler=recover,
)
```

An explicit per-node value wins. Defaults do not flow into subgraphs. Retry
and timeout defaults also apply to handler nodes; cache and error-handler
defaults apply only to regular nodes.

## Execution metadata and drain signals

Nodes can inspect execution identity and retry state through Python
`runtime.execution_info` or JavaScript `runtime.executionInfo`. These include
thread, run, checkpoint, task, attempt number, and first-attempt time.

Server deployments additionally expose assistant, graph, and authenticated
user data through `server_info` or `serverInfo`; this data is absent in local
execution. These surfaces require Python `langgraph>=1.1.5` or JavaScript
`@langchain/langgraph>=1.2.8`.

With Python `langgraph>=1.2`, `runtime.drain_requested` and
`runtime.drain_reason` become available after a run drain is requested. A node
can skip expensive work before execution reaches the next superstep boundary:

```python
def node(state: State, runtime: Runtime):
    if runtime.drain_requested:
        return {"status": "skipped", "reason": runtime.drain_reason}
    return {"status": do_work()}
```

## Fan-in

In Python, `add_node(..., defer=True)` postpones the node until all pending
tasks finish. Use it as a final fan-in point when parallel branches have
different lengths:

```python
builder.add_node("finalize", finalize, defer=True)
```

## Recursion budget

Since LangGraph Python 1.0.6, the default recursion limit is 1000 super-steps.
JavaScript defaults to 25. Set `recursion_limit` or `recursionLimit` as a
top-level invocation configuration key, never under `configurable`.

```python
result = graph.invoke(inputs, config={"recursion_limit": 100})
current_step = config["metadata"]["langgraph_step"]
```

Nodes can read the current step from `metadata.langgraph_step`. Python graphs
can declare a `RemainingSteps` managed state value and route proactively
before exhausting the budget.

## Graph migrations with checkpoints

Completed threads tolerate arbitrary topology changes. An interrupted thread
cannot safely resume if its nodes have been renamed or removed.

State keys may be added or removed compatibly. Renaming a key loses its saved
value, and incompatible type changes can break state loaded from older
threads.
