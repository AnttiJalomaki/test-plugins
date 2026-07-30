# Agents and workflows

Use this guide when selecting an agent, adapting tools, consuming a run,
coordinating events, or deciding where workflow data belongs.

## Choose an agent by its action interface

`FunctionAgent` requires an LLM with compatible native function or tool calling.
When native calls are unavailable, `ReActAgent` uses text-based reasoning and
action parsing. `CodeActAgent` is the option for code-action scenarios.

Current agents accept ordinary synchronous and asynchronous callables as tools.
Their type hints and docstrings provide the tool schema:

```python
from llama_index.core.agent.workflow import FunctionAgent

def multiply(a: float, b: float) -> float:
    """Multiply two numbers."""
    return a * b

agent = FunctionAgent(tools=[multiply], llm=llm)
```

Use `FunctionTool` when a callable needs explicit metadata or adaptation rather
than the automatically derived schema.

## Consume the run handler correctly

Calls to `agent.run(...)` and `workflow.run(...)` return awaitable workflow
handlers. They do not return completed results immediately. Keep the handler so
the same run can provide both live events and its eventual result:

```python
handler = agent.run("What is 12 times 34?")
async for event in handler.stream_events():
    ...
result = await handler
```

Do not start a second run merely to obtain the final value after streaming.

## Create memory explicitly

Construct current agent memory with `Memory.from_defaults` and pass it into the
run:

```python
from llama_index.core.memory import Memory

memory = Memory.from_defaults(session_id="session-123", token_limit=40000)
response = await agent.run("...", memory=memory)
```

Conversation history and memory blocks belong to `Memory`. They are distinct
from workflow `Context`, which carries execution state and events for a run.

## Coordinate fan-out and fan-in with events

For finite fan-out known by a step, return a list of events. For dynamic work,
emit events through `Context.send_event`. At fan-in, use
`Context.collect_events` to wait for the expected event set.

Concurrent results do not necessarily arrive in input order. If output order
matters, carry an ordering key in the event data and restore order explicitly.

## Separate serializable state from resources

Put per-run serializable state in the asynchronous `ctx.store` interface using
its get, set, and edit operations. This is the state suitable for workflow
execution and checkpointing.

Clients, indexes, models, and configuration are external resources. Supply them
through workflow resources, whose factories and validation provide dependency
injection. Do not put live resource objects into checkpointable workflow state.

## Validate the typed event graph

Run validation in a test or at application startup:

```python
workflow = RagFlow(timeout=60)
workflow.validate()
```

`workflow.validate()` checks the typed step graph for:

- Start and stop paths.
- Produced events with no consumer.
- Consumed events with no producer.
- Dead ends.

Validation catches structural mismatches before a live execution reaches the
affected branch. It complements tests of handler streaming, state changes,
resource injection, and concurrent fan-in behavior.
