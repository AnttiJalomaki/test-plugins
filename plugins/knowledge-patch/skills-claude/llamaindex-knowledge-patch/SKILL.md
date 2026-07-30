---
name: llamaindex-knowledge-patch
description: LlamaIndex
version: null
license: MIT
metadata:
  author: Nevaberry
---


# LlamaIndex Knowledge Patch

Use this skill when maintaining LlamaIndex applications that use core indexes,
ingestion pipelines, agents, or workflows. Start with the migration rules below,
then open the topic guide that matches the code being changed.

## Reference index

| Reference | Topics |
| --- | --- |
| [API migrations](references/api-migrations.md) | `ServiceContext` and `LLMPredictor` removal, Pydantic v2, package coordination, Python support, workflow migration |
| [Ingestion and storage](references/ingestion-storage.md) | `IngestionPipeline`, cache and docstore state, parallel execution, document updates, persisted and external vector stores |
| [Agents and workflows](references/agents-workflows.md) | Agent selection, tools, run handlers, memory, concurrency, state, resources, validation |

## Migration priorities

### Replace removed global abstractions

Do not build new code around `ServiceContext` or `LLMPredictor`. Configure
process-wide defaults through `Settings`, or pass the LLM, embedding model, and
transformations directly when multiple configurations must coexist.

```python
from llama_index.core import Settings

Settings.llm = llm
Settings.embed_model = embed_model
```

Treat this as more than an import edit. Audit custom readers, nodes, output
parsers, tools, models, validators, serialization, and integrations for
Pydantic v2 behavior.

### Upgrade the package set coherently

Resolve the entire `llama-index-*` environment together. Integration packages
have independent versions and core constraints, so neither upgrading core alone
nor pinning every distribution to `0.12.0` is a safe upgrade strategy.

```python
from importlib.metadata import version

core_version = version("llama-index-core")
starter_version = version("llama-index")
```

Read each selected release's Python metadata. Python 3.8 is no longer supported,
and some newer releases may require a higher interpreter version. Preserve the
resolved lockfile after the upgrade.

### Redesign old agent entry points

Use workflow-based agents from `llama_index.core.agent.workflow`. Their runs are
asynchronous workflow handlers with event streams; older `OpenAIAgent`, runner,
and worker examples do not describe this execution model.

```python
from llama_index.core.agent.workflow import (
    AgentWorkflow,
    FunctionAgent,
    ReActAgent,
)
```

Use `Memory` for current agent memory. `ChatMemoryBuffer`,
`ChatSummaryMemoryBuffer`, and `VectorMemory` are deprecated.

### Redesign QueryPipeline graphs as workflows

A workflow is not a renamed `QueryPipeline`. Re-express the control flow with
typed Pydantic events, asynchronous `@step` methods, branches or loops,
`Context`, streaming, and checkpoint-aware execution.

Core applications can keep the `llama_index.core.workflow` import surface.
Standalone workflow applications can install `llama-index-workflows` and import
from `workflows`.

## Ingestion quick reference

### Own updates with an ingestion pipeline

`VectorStoreIndex.from_documents` is convenient, but use `IngestionPipeline`
when the application needs transformation caching, document update policies, or
direct vector-store insertion.

When the embedding model or its configuration changes, rebuild or explicitly
verify persisted vector indexes. Exercise ingestion, retrieval, agent, and
workflow paths together during an upgrade.

### Keep persisted components explicit

`index.storage_context.persist()` saves local stores. It does not necessarily
back up an external vector store, and a remote collection alone may not contain
node text, mappings, or document state. Reloading also does not prove that the
remote collection exists or that its embeddings are compatible.

```python
from llama_index.core import StorageContext, load_index_from_storage

index.storage_context.persist(persist_dir="./storage")
storage_context = StorageContext.from_defaults(persist_dir="./storage")
index = load_index_from_storage(storage_context)
```

If a persist directory contains multiple indexes, select the expected index ID.

### Separate cache state from document state

Ingestion cache reuse is based on each node and transformation combination.
Custom transformation changes can reuse stale output when stable serialized
settings or hashes do not change.

```python
pipeline.persist("./pipeline-state")
pipeline.load("./pipeline-state")
```

The saved pipeline cache is distinct from docstore state used to detect
duplicate, updated, and deleted documents.

### Choose async and worker execution deliberately

Use `pipeline.arun()` in asynchronous code. `num_workers` enables process-based
parallelism for suitable synchronous transformations; provider clients and
non-picklable transformations can make it invalid or slower.

```python
nodes = await pipeline.arun(documents=documents, num_workers=4)
```

### Protect partial crawls from deletion

With both a docstore and vector store, stable `document.doc_id` or
`node.ref_doc_id` values anchor hash-based duplicate and update detection.
Deletion strategies that remove documents absent from a run require an
authoritative, complete input inventory. Do not apply them to a partial crawl.

## Agents quick reference

### Match the agent to the tool interface

Choose `FunctionAgent` only when the LLM supports compatible native function or
tool calls. Use `ReActAgent` for text-based reasoning and action parsing when it
does not; use `CodeActAgent` for code-action scenarios.

Plain synchronous and asynchronous callables can be tools. Type hints and
docstrings supply their schemas; use `FunctionTool` when metadata or adaptation
must be explicit.

```python
from llama_index.core.agent.workflow import FunctionAgent

def multiply(a: float, b: float) -> float:
    """Multiply two numbers."""
    return a * b

agent = FunctionAgent(tools=[multiply], llm=llm)
```

### Retain and await the run handler

`agent.run(...)` and `workflow.run(...)` return streamable awaitables, not
completed results. Retain one handler, consume its live events if needed, then
await that same handler for the final result.

```python
handler = agent.run("What is 12 times 34?")
async for event in handler.stream_events():
    ...
result = await handler
```

### Keep memory, state, and resources separate

Create `Memory` explicitly and pass it to `run`. Conversation history and memory
blocks are different from workflow `Context`, which carries per-run execution
state and events.

```python
from llama_index.core.memory import Memory

memory = Memory.from_defaults(session_id="session-123", token_limit=40000)
response = await agent.run("...", memory=memory)
```

Store serializable per-run values through asynchronous `ctx.store` operations.
Inject clients, indexes, models, and configuration as workflow resources rather
than treating live objects as checkpointable state.

### Use events for dynamic concurrency

A step can return a list of events for finite fan-out. For dynamic fan-out, emit
work with `Context.send_event`; at fan-in, use `Context.collect_events` for the
expected event set. Never assume results arrive in input order.

### Validate the event graph

Call `workflow.validate()` in tests or startup checks. It detects missing
start/stop paths, produced events without consumers, consumed events without
producers, and dead ends.

```python
workflow = RagFlow(timeout=60)
workflow.validate()
```

## Working method

1. Identify whether the change affects API migration, ingestion/storage, or
   agent/workflow execution.
2. Apply the matching quick-reference rule before adapting local code.
3. Open the corresponding reference guide for edge cases and lifecycle details.
4. Test the complete affected path, including persisted or remote state where
   relevant.
