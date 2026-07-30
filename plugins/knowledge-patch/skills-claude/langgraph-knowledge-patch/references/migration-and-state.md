# Migration and state

## Agent and package migration

The `langgraph-v1` migration moves agent construction to LangChain while
retaining LangGraph as the execution runtime and adding middleware support.

### Agent factory

Import `create_agent` from `langchain.agents` in Python or `createAgent` from
`langchain` in JavaScript. Rename the prompt argument to `system_prompt` or
`systemPrompt`, respectively.

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

### Python prebuilt replacements

Use these replacements when removing deprecated prebuilt APIs:

| Deprecated surface | Replacement |
| --- | --- |
| LangGraph `AgentState` import | `AgentState` from `langchain.agents` |
| Pydantic or structured-response state variants | `AgentState` |
| `HumanInterruptConfig`, `ActionRequest` | `InterruptOnConfig` |
| `HumanInterrupt` | `HITLRequest` |
| `ValidationNode` | Automatic tool-input validation in `create_agent` |
| `MessageGraph` | `StateGraph` with a `messages` key |

### Runtime and package output

JavaScript LangGraph packages require Node.js 22 or newer. The Python-side
LangChain packages require Python 3.10 or newer.

JavaScript packages now publish bundled builds instead of raw TypeScript.
Imports into private paths under `dist/` are unsupported; import from public
package modules.

## JavaScript state schemas

`StateSchema` is the recommended state API. It combines standard field schemas
with LangGraph value types:

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

`MessagesValue` provides message-aware reduction. `ReducedValue` takes the
field schema and default plus a custom reducer. `Annotation.Root` and direct
Zod v3/v4 integrations remain legacy alternatives.

## Python Pydantic validation boundary

A Pydantic `BaseModel` may define graph state, but validation occurs only on
input to the first node. Later node updates and final graph output are not
validated through that model. The first node receives the validated model;
`invoke` still returns a dictionary.

For message fields that must serialize over the wire, use `AnyMessage` rather
than `BaseMessage`.

## Private and runtime-only state

### Private channels remain visible in snapshots

A node's input schema controls what it can read, not which channels it may
update. Schemas declared by nodes can add private channels to the union of
graph state.

Input, output, and private schemas do not redact `values` streams. When a v3
event stream must exclude private channels, filter snapshots with
`output_keys` in Python or `outputKeys` in JavaScript:

```python
stream = graph.stream_events(
    {"user_input": "My"},
    version="v3",
    output_keys=["graph_output"],
)
```

### Bypass a reducer once

In Python, wrap a replacement in `Overwrite` to bypass the channel's reducer
for one update:

```python
from langgraph.types import Overwrite

return {"items": Overwrite(["replacement"])}
```

### Exclude JavaScript values from checkpoints

`UntrackedValue` holds execution-time state that starts fresh after resume and
is excluded from checkpoints. Use it for connections, temporary caches, and
similar runtime-only objects.

```typescript
const State = new StateSchema({
  dbConnection: new UntrackedValue<DatabaseConnection>(),
  tempCache: new UntrackedValue(z.record(z.string(), z.unknown()), {
    guard: false,
  }),
});
```

Its default `guard: true` rejects multiple same-step writes. With
`guard: false`, multiple writes are accepted and the last value wins.

## Runtime context

Runtime context carries dependencies and invocation data without making them
graph state.

In Python, declare `context_schema`, accept `Runtime[Context]`, read
`runtime.context`, and pass the invocation's context with `context=`:

```python
from langgraph.graph import END, START, StateGraph
from langgraph.runtime import Runtime
from typing_extensions import TypedDict

class State(TypedDict):
    tenant: str

class Context(TypedDict):
    tenant: str

def node(state: State, runtime: Runtime[Context]):
    return {"tenant": runtime.context["tenant"]}

graph = (
    StateGraph(State, context_schema=Context)
    .add_node("node", node)
    .add_edge(START, "node")
    .add_edge("node", END)
    .compile()
)
graph.invoke({}, context={"tenant": "acme"})
```

In JavaScript, pass a context schema as the second `StateGraph` constructor
argument, read `runtime.context`, and supply `{ context: ... }` in invocation
options.
