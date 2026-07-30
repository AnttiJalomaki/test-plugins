# Persistence and Subgraphs

Source batches: `persistence-and-checkpointing`, `subgraphs`.

## Checkpoint namespaces

Every checkpoint contains `checkpoint_ns`. The root namespace is `""`; a
subgraph uses `"node_name:uuid"`; nested subgraph segments are joined with
`|`.

```python
def node(state, config):
    namespace = config["configurable"]["checkpoint_ns"]
```

A subgraph update may not immediately be visible to the parent. Use a shared
Store, or arrange for the subgraph to write to the parent checkpoint, when data
must cross the namespace boundary.

## Durability modes

Execution methods accept three per-run durability modes:

- `durability="exit"` writes when execution completes, errors, or interrupts.
- `durability="async"` writes while the next step executes and can lose the
  newest checkpoint in a crash.
- `durability="sync"` persists before advancing to the next step.

```python
graph.stream({"input": "test"}, durability="sync")
```

## Delta-backed channels

Python `langgraph>=1.2` adds the beta `DeltaChannel` for append-heavy state.
Instead of storing the complete accumulated channel in every checkpoint, it
stores incremental writes and reconstructs the value from the nearest
`_DeltaSnapshot` and its descendant writes.

A saver used with deltas must support exact lookup by
`(thread_id, checkpoint_ns, checkpoint_id)`. Pruning must retain the required
write chain back to a snapshot or first materialize a new snapshot. Copying a
thread must copy ancestors back to a snapshot for every delta channel.

## Custom checkpoint-saver contract

A `BaseCheckpointSaver` implements `put`, `put_writes`, `get_tuple`, `list`,
and `delete_thread`, or their asynchronous counterparts.

The implementation must:

- Support both exact-checkpoint-ID and latest-checkpoint reads.
- Return history newest first and honor `before` and `limit`.
- Delete both checkpoint rows and write rows for a thread.
- Pass persisted checkpoints, writes, and complete metadata through
  `self.serde`.
- Use `WRITES_IDX_MAP` for reserved channels including `__error__` and
  `__interrupt__`.

The `langgraph-checkpoint-conformance` package exercises every base method and
detected extension, including delta history, so custom saver implementations
can enforce compatibility in CI.

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

`JsonPlusSerializer` normally uses msgpack and JSON. Set
`pickle_fallback=True` only when state contains unsupported values such as
dataframes.

Wrap any saver with `EncryptedSerializer.from_pycryptodome_aes()` to encrypt
persisted state. It reads `LANGGRAPH_AES_KEY` unless passed a key explicitly.

```python
from langgraph.checkpoint.serde.encrypted import EncryptedSerializer
from langgraph.checkpoint.postgres import PostgresSaver

serde = EncryptedSerializer.from_pycryptodome_aes()
saver = PostgresSaver.from_conn_string("postgresql://...", serde=serde)
saver.setup()
```

`PostgresSaver` and `AsyncPostgresSaver` require `thread_id` strings shorter
than 255 characters. Use a UUID or a hash rather than a long deterministic
identifier.

```python
import uuid

config = {"configurable": {"thread_id": str(uuid.uuid4())}}
```

## Subgraph state lifetimes

The subgraph's compile-time `checkpointer` value determines its lifetime:

- The default `None` starts fresh on every call but inherits the parent
  checkpointer for interrupts and durable execution during that call.
- `True` retains state across calls on the same thread.
- `False` disables checkpointing, interrupts, durable execution, and state
  inspection.

The parent graph needs its own checkpointer for either stateful mode to work.

```python
per_call = builder.compile()
per_thread = builder.compile(checkpointer=True)
stateless = builder.compile(checkpointer=False)

graph = parent_builder.compile(checkpointer=MemorySaver())
```

## Per-thread concurrency

Two concurrent calls to the same `checkpointer=True` subgraph target the same
checkpoint namespace and conflict. Serialize access. For a tool-wrapped agent,
a run limit is one way to enforce it. Use per-invocation persistence when calls
should remain independent.

```python
middleware = [
    ToolCallLimitMiddleware(tool_name="ask_expert", run_limit=1),
]
```

## Stable child namespaces

Persistent subgraphs invoked inside one node receive namespaces according to
call order. Reordering several children can therefore cause one child to load
another child's saved state.

Wrap each child in a `StateGraph` node with a unique name to establish a stable
namespace. A subgraph passed directly to `add_node` already receives a
name-based namespace.

```python
def named_child(agent, name):
    return (
        StateGraph(MessagesState)
        .add_node(name, agent)
        .add_edge("__start__", name)
        .compile()
    )
```

## State-inspection discovery boundary

`get_state(..., subgraphs=True)` exposes a child snapshot only when LangGraph
can discover that child statically: it was added as a node or invoked directly
inside a node. A child hidden behind a tool or another indirection is not
discoverable for this inspection.

Per-invocation state is inspectable only for the current interrupted call.
Per-thread state accumulates. Stateless children have no snapshot.

```python
snapshot = graph.get_state(config, subgraphs=True)
child_state = snapshot.tasks[0].state
```
