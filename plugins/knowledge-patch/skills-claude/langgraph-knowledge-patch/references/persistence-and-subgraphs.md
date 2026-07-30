# Persistence and subgraphs

## Checkpoint namespaces

Every checkpoint has a `checkpoint_ns`. The root graph uses `""`, a subgraph
uses `"node_name:uuid"`, and nested subgraph namespaces join segments with
`|`.

```python
def node(state, config):
    namespace = config["configurable"]["checkpoint_ns"]
```

A subgraph update may not be immediately visible to its parent. When data must
cross the boundary, use a shared Store or arrange for the subgraph to write
into the parent checkpoint.

## Durability modes

Execution methods accept a per-run `durability`:

| Mode | Persistence point | Tradeoff |
| --- | --- | --- |
| `exit` | Completion, error, or interrupt | No intermediate writes |
| `async` | While the next step runs | Can lose the newest checkpoint in a crash |
| `sync` | Before advancing | Stronger persistence before progress |

```python
graph.stream({"input": "test"}, durability="sync")
```

## Delta-backed state

With Python `langgraph>=1.2`, the beta `DeltaChannel` optimizes append-heavy
state. Instead of copying the full accumulated channel into every checkpoint,
it saves incremental writes and reconstructs values from the nearest
`_DeltaSnapshot` and its ancestor writes.

A custom saver used with delta channels must support exact lookup by
`(thread_id, checkpoint_ns, checkpoint_id)`. Pruning must keep the write chain
needed to reach a snapshot or first materialize a snapshot. Copying a thread
must include ancestors back to a snapshot for every delta channel.

## Custom checkpoint saver contract

A `BaseCheckpointSaver` implements `put`, `put_writes`, `get_tuple`, `list`,
and `delete_thread`, or the asynchronous counterparts.

The backend must:

- Support both exact-checkpoint-ID and latest-checkpoint reads.
- Return history newest first, honoring `before` and `limit`.
- Delete both checkpoint rows and write rows for a thread.
- Serialize persisted checkpoints, writes, and complete metadata through
  `self.serde`.
- Use `WRITES_IDX_MAP` for reserved channels such as `__error__` and
  `__interrupt__`.

The `langgraph-checkpoint-conformance` package checks the base methods and
detected extensions, including delta history:

```python
import asyncio

from langgraph.checkpoint.conformance import checkpointer_test, validate

@checkpointer_test(name="MyCheckpointer")
async def my_checkpointer():
    async with MyCheckpointer.create() as saver:
        yield saver

async def main():
    report = await validate(my_checkpointer)
    if not report.passed_all_base():
        raise RuntimeError("checkpointer failed conformance")

asyncio.run(main())
```

## Serialization and encryption

`JsonPlusSerializer` normally uses msgpack and JSON. Enable
`pickle_fallback=True` only for unsupported values such as dataframes.

Any saver can encrypt persisted state with
`EncryptedSerializer.from_pycryptodome_aes()`. It reads `LANGGRAPH_AES_KEY`
unless the caller supplies a key:

```python
from langgraph.checkpoint.serde.encrypted import EncryptedSerializer
from langgraph.checkpoint.postgres import PostgresSaver

serde = EncryptedSerializer.from_pycryptodome_aes()
saver = PostgresSaver.from_conn_string("postgresql://...", serde=serde)
saver.setup()
```

`PostgresSaver` and `AsyncPostgresSaver` require `thread_id` values shorter
than 255 characters. Use a UUID or hash for an oversized deterministic ID:

```python
import uuid

config = {"configurable": {"thread_id": str(uuid.uuid4())}}
```

## Subgraph checkpointer modes

The checkpointer value used to compile a subgraph determines state lifetime:

| Setting | Behavior |
| --- | --- |
| Default `None` | Fresh state on every call; inherits the parent checkpointer for interrupts and durable execution within that call |
| `True` | Retains state across calls on the same thread |
| `False` | Disables checkpointing, interrupts, durable execution, and state inspection |

```python
per_call = builder.compile()
per_thread = builder.compile(checkpointer=True)
stateless = builder.compile(checkpointer=False)

graph = parent_builder.compile(checkpointer=MemorySaver())
```

The parent graph must have a checkpointer for either stateful mode to work.

## Concurrency and stable child identity

Two concurrent calls to the same `checkpointer=True` subgraph share its
checkpoint namespace and conflict. Serialize them. For a tool-wrapped agent,
a run limit is one option:

```python
middleware = [
    ToolCallLimitMiddleware(tool_name="ask_expert", run_limit=1),
]
```

Use per-invocation persistence when calls should remain independent.

Persistent subgraphs invoked inside one node receive namespaces according to
call order. Reordering multiple children can cause one child to load another
child's state. Wrap every child in a `StateGraph` node with a unique name to
give it a stable namespace:

```python
def named_child(agent, name):
    return (
        StateGraph(MessagesState)
        .add_node(name, agent)
        .add_edge("__start__", name)
        .compile()
    )
```

A subgraph passed directly to `add_node` already receives a name-based
namespace.

## State inspection boundary

`get_state(..., subgraphs=True)` exposes a child's snapshot only when the
child is statically discoverable: it is added as a node or invoked directly
inside a node. A child hidden behind a tool or other indirection is not
discoverable.

Per-invocation child state can be inspected only during the current
interrupted call. Per-thread state accumulates across calls. A stateless child
has no snapshot.

```python
snapshot = graph.get_state(config, subgraphs=True)
child_state = snapshot.tasks[0].state
```
