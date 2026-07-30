# Migrations and State

Source batches: `langgraph-v1`, `graph-api-overview`, `graph-api-usage`.

## Agent construction and deprecated prebuilts

LangGraph v1 deprecates its React-agent prebuilt in favor of LangChain's agent
factory. The replacement still runs on LangGraph and adds middleware.

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

The import now comes from `langchain`. Rename `prompt` to `system_prompt` in
Python and to `systemPrompt` in JavaScript.

For Python prebuilt APIs:

- Import `AgentState` from `langchain.agents`.
- The Pydantic and structured-response state variants collapse into
  `AgentState`.
- `HumanInterruptConfig` and `ActionRequest` become `InterruptOnConfig`.
- `HumanInterrupt` becomes `HITLRequest`.
- `ValidationNode` is replaced by automatic tool-input validation in
  `create_agent`.
- Replace `MessageGraph` with a `StateGraph` whose state contains a `messages`
  key.

## Runtime and public import requirements

JavaScript LangGraph packages require Node.js 22 or newer. Python-side
LangChain packages require Python 3.10 or newer.

JavaScript packages now ship bundled builds instead of raw TypeScript output.
Imports that reach into `dist/` are private and must be replaced with public
module imports.

## JavaScript state schemas

Use `StateSchema` for new JavaScript graphs. It accepts ordinary field schemas
and LangGraph value types. `MessagesValue` provides message-aware reduction;
`ReducedValue` combines a schema and default with a reducer.

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

`Annotation.Root` and the direct Zod v3/v4 integrations are legacy
alternatives.

## Python Pydantic validation boundaries

A Python `BaseModel` may be the graph state schema, but LangGraph validates it
only as input to the first node. The first node receives the validated model;
later node updates and graph output are not validated through that model, and
`invoke` returns a dictionary.

Use `AnyMessage` rather than `BaseMessage` when a message field must serialize
over the wire.

## Runtime context

Put invocation-scoped dependencies and configuration in runtime context
instead of graph state. Python graphs declare `context_schema`, nodes read
typed data from `Runtime.context`, and callers pass `context=`.

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

## State visibility and replacement

A node's input schema restricts the fields it reads, not the graph channels it
may update. Schemas declared by nodes can add private channels to the union of
graph state.

Input, output, and private schemas are not redaction boundaries for `values`
streams. When private channels must not appear in snapshots, use a v3 event
stream and filter it with `output_keys` in Python or `outputKeys` in
JavaScript.

```python
stream = graph.stream_events(
    {"user_input": "My"},
    version="v3",
    output_keys=["graph_output"],
)
```

In Python, wrap a value in `Overwrite` to bypass its channel reducer for a
single update:

```python
from langgraph.types import Overwrite

return {"items": Overwrite(["replacement"])}
```

JavaScript `UntrackedValue` stores execution-time values outside checkpoints,
so they start fresh after a resume. It is appropriate for connections,
temporary caches, and similar runtime-only objects. The default `guard: true`
rejects multiple writes in one step; `guard: false` permits them and retains
the last value.

```typescript
const State = new StateSchema({
  dbConnection: new UntrackedValue<DatabaseConnection>(),
  tempCache: new UntrackedValue(z.record(z.string(), z.unknown()), {
    guard: false,
  }),
});
```

## State and topology migrations

Completed threads tolerate arbitrary graph-topology changes. Interrupted
threads cannot safely resume after a node on their pending path is renamed or
removed.

State keys may be added or removed compatibly. Renaming a key discards access
to its saved value, and an incompatible type change can make older thread
state unusable.
