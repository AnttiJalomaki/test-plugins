# Agents and Workflows

## Select an Agent by Tool Interface

Choose the agent from the LLM's interaction contract:

| Contract | Agent |
| --- | --- |
| Compatible native function or tool calling | `FunctionAgent` |
| Text-based reasoning and action parsing | `ReActAgent` |
| Executable code actions | `CodeActAgent` |

Do not select `FunctionAgent` merely because the application has Python
functions. The configured LLM must expose a compatible native tool-calling
interface. `ReActAgent` is the text-protocol option when native calls are not
available.

Current agents accept plain synchronous or asynchronous callables as tools.
Their type hints and docstrings provide schema information:

```python
from llama_index.core.agent.workflow import FunctionAgent

def multiply(a: float, b: float) -> float:
    """Multiply two numbers."""
    return a * b

agent = FunctionAgent(tools=[multiply], llm=llm)
```

Use `FunctionTool` when a tool needs explicit metadata, a curated schema, or
adaptation instead of automatic callable inspection.

## Run Handlers Are Streamable Awaitables

`agent.run(...)` and `workflow.run(...)` return workflow handlers. Starting a
run does not immediately return its final response. Retain the handler so the
same execution can provide live events and its eventual result:

```python
handler = agent.run("What is 12 times 34?")
async for event in handler.stream_events():
    ...
result = await handler
```

Do not start a second run to obtain the final value after streaming the first.
Await the same handler.

## Create Agent Memory Explicitly

Use `Memory.from_defaults` and pass the memory to the run:

```python
from llama_index.core.memory import Memory

memory = Memory.from_defaults(
    session_id="session-123",
    token_limit=40000,
)
response = await agent.run("...", memory=memory)
```

Memory stores conversation history and memory blocks. It is not workflow
`Context`. Context carries per-run execution state and event coordination.
Keep session memory lifetime separate from workflow-run lifetime.

`ChatMemoryBuffer`, `ChatSummaryMemoryBuffer`, and `VectorMemory` are
deprecated and should not anchor new designs.

## Fan-Out and Fan-In

For a known finite fan-out, a step can return a list of events. For dynamically
discovered work, emit one event per item with `Context.send_event`.

Fan-in uses `Context.collect_events` to wait for the expected event set. Event
arrival follows completion timing, so result order is not necessarily input
order. Carry a stable key or ordinal in each event when output order matters,
then reorder explicitly after collection.

Design fan-in expectations carefully. Waiting for an event type or count that
no branch can produce leaves the workflow unable to complete.

## State and Resources

Serializable per-run state belongs in `ctx.store`, using its asynchronous
get/set/edit interface. Use it for values that participate in a run and may
need to survive checkpointing.

Live external objects belong in workflow resources:

- API and database clients;
- indexes and vector-store connections;
- LLM and embedding instances;
- configuration and services.

Resource factories and validation support dependency injection. Recreate live
resources when resuming execution rather than treating open clients or other
runtime objects as checkpointable state.

The boundary is:

| Concern | Location |
| --- | --- |
| Serializable value scoped to a run | `ctx.store` |
| Conversation history or memory blocks | `Memory` |
| Event coordination | workflow `Context` |
| Client, index, model, or configuration dependency | workflow resource |

## Validate the Event Graph

Validate workflows in tests or at application startup:

```python
workflow = RagFlow(timeout=60)
workflow.validate()
```

Validation inspects the typed step graph for:

- viable start and stop paths;
- produced events with no consumers;
- consumed events with no producers;
- dead ends.

Validation catches structural event-graph errors before a representative run,
but it does not replace execution tests for dynamic branches, resource
failures, fan-in counts, streaming, or checkpoint resumption.

## Workflow Architecture

Workflows express control flow through typed Pydantic events and asynchronous
`@step` methods. Branches, loops, streaming, `Context`, and checkpoints are
first-class design concerns. A legacy `QueryPipeline` DAG therefore needs a
behavioral redesign rather than a class rename.

Use `llama_index.core.workflow` in core applications. A standalone workflow
application can install `llama-index-workflows` and import from `workflows`.

## Test Matrix

At minimum, test:

1. each selected agent against the configured LLM tool interface;
2. automatic callable schema generation for sync and async tools;
3. explicit `FunctionTool` metadata where used;
4. event streaming and awaiting the same handler;
5. memory reuse across intended conversation turns;
6. dynamic fan-out, fan-in, and order-independent completion;
7. `ctx.store` serialization and edits;
8. resource construction and validation;
9. static workflow validation;
10. checkpoint and resume behavior for durable flows.
