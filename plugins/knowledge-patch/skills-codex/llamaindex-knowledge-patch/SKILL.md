---
name: llamaindex-knowledge-patch
description: LlamaIndex
version: null
license: MIT
metadata:
  author: Nevaberry
---


# LlamaIndex Knowledge Patch

Use this skill for LlamaIndex migrations, package compatibility, ingestion,
persistence, workflow-based agents, explicit memory, event concurrency,
durable state, resources, and validation.

Prefer project manifests, installed package metadata, application code, and
tests when they disagree with generic guidance. LlamaIndex integrations have
independent distribution versions, so inspect the actual environment before
changing imports or constraints.

## Reference Index

| Reference | Topics |
| --- | --- |
| [Migration and packaging](references/migration-and-packaging.md) | Removed APIs, Pydantic v2, coordinated dependency upgrades, Python support, migration verification |
| [Ingestion and storage](references/ingestion-and-storage.md) | `IngestionPipeline`, cache identity, update strategies, async/process execution, local and remote persistence |
| [Agents and workflows](references/agents-and-workflows.md) | Agent selection, tools, awaitable handlers, memory, workflow events, concurrency, state, resources, validation |

## Breaking Changes First

### Replace removed global service abstractions

Do not build current code around `ServiceContext` or `LLMPredictor`. Configure
process-wide defaults through `Settings`, or pass components directly when
different indexes and pipelines require different configurations.

```python
from llama_index.core import Settings

Settings.llm = llm
Settings.embed_model = embed_model
```

Prefer direct injection for libraries, tests, multi-tenant services, and any
process that hosts more than one configuration.

Audit custom readers, nodes, output parsers, tools, validators, serializers,
and integrations for Pydantic v2 compatibility. Do not assume that replacing
the removed constructors completes the migration.

### Upgrade the package family coherently

Treat a core upgrade as a dependency-resolution operation across all installed
`llama-index-*` distributions. Integration packages retain independent
versions and core constraints; assigning one release number to every package
can produce an invalid environment.

```python
from importlib.metadata import version

core_version = version("llama-index-core")
starter_version = version("llama-index")
```

Resolve the whole environment, run integration tests, and preserve the
resulting lockfile. Check each selected distribution's Python requirement:
Python 3.8 is not supported by the coordinated v0.12 package generation, and
newer individual releases may require a later interpreter.

### Migrate legacy agents to workflow agents

Import current agents from `llama_index.core.agent.workflow`:

```python
from llama_index.core.agent.workflow import (
    AgentWorkflow,
    FunctionAgent,
    ReActAgent,
)
```

Current agent runs are asynchronous workflow executions. Replace legacy
provider-specific agent classes, runner/worker patterns, and implicit memory
assumptions rather than only changing imports.

### Redesign QueryPipeline graphs

Workflows are not a renamed `QueryPipeline`. Translate the behavior into:

- typed Pydantic events;
- asynchronous `@step` methods;
- explicit branches, loops, and stop paths;
- `Context` state and emitted events;
- streamed results and checkpoint-aware execution.

Applications already based on core can use `llama_index.core.workflow`.
Standalone workflow applications can install `llama-index-workflows` and
import from `workflows`.

## Ingestion Quick Reference

### Choose the maintainable ingestion surface

`VectorStoreIndex.from_documents` is convenient for simple construction.
Choose `IngestionPipeline` when the application needs transformation caching,
repeatable document update behavior, or direct insertion into a vector store.

```python
pipeline = IngestionPipeline(
    transformations=transformations,
    docstore=docstore,
    vector_store=vector_store,
)
nodes = pipeline.run(documents=documents)
```

When an embedding implementation or configuration changes, rebuild or
explicitly verify persisted vector indexes. An index that reloads successfully
does not prove that stored vectors are compatible with the current embedder.

### Keep cache and document state distinct

The ingestion cache keys each node together with each transformation. A custom
transformation whose behavior changes without changing stable serialized
settings or hashes can incorrectly reuse old output.

```python
pipeline.persist("./pipeline-state")
pipeline.load("./pipeline-state")
```

This pipeline cache is separate from the docstore state used to identify
duplicates, updates, and deletions. Persist and restore the state required by
each concern.

### Protect partial crawls from deletion

Stable `document.doc_id` and `node.ref_doc_id` values underpin document hashes
and update detection. A `DocstoreStrategy` that deletes documents absent from
the current run is safe only when that run is an authoritative full inventory.
Do not use absence-based deletion for a partial crawl.

### Pick an execution mode deliberately

```python
nodes = await pipeline.arun(
    documents=documents,
    num_workers=4,
)
```

Use `arun()` in asynchronous applications. `num_workers` uses process-based
parallelism for suitable synchronous transformations. Provider clients,
non-picklable transformations, startup overhead, and small workloads can make
process workers invalid or slower.

## Persistence Quick Reference

`index.storage_context.persist()` saves local stores. With an external vector
store, the directory may contain only auxiliary state, while remote vectors
may omit node text, mappings, or document state. Reload neither verifies the
remote collection nor establishes embedding compatibility. A directory
containing multiple indexes may also require an explicit index ID.

```python
from llama_index.core import StorageContext, load_index_from_storage

index.storage_context.persist(persist_dir="./storage")
storage_context = StorageContext.from_defaults(persist_dir="./storage")
index = load_index_from_storage(storage_context)
```

Back up and validate the local stores and the remote collection as one logical
system.

## Agent Selection Quick Reference

| Need | Agent |
| --- | --- |
| LLM has compatible native function/tool calling | `FunctionAgent` |
| LLM needs text-based reasoning/action parsing | `ReActAgent` |
| Task centers on executable code actions | `CodeActAgent` |

Current agents accept plain synchronous or asynchronous Python callables.
Type hints and docstrings supply their tool schemas. Use `FunctionTool` when
metadata must be explicit or a callable needs adaptation.

```python
def multiply(a: float, b: float) -> float:
    """Multiply two numbers."""
    return a * b

agent = FunctionAgent(tools=[multiply], llm=llm)
```

## Running and Streaming

`agent.run(...)` and `workflow.run(...)` return awaitable workflow handlers,
not completed results. Keep one handler, stream from it, and await that same
handler for the final result:

```python
handler = agent.run("What is 12 times 34?")
async for event in handler.stream_events():
    ...
result = await handler
```

Create current agent memory explicitly and pass it to the run:

```python
from llama_index.core.memory import Memory

memory = Memory.from_defaults(
    session_id="session-123",
    token_limit=40000,
)
response = await agent.run("...", memory=memory)
```

Memory owns conversation history and memory blocks. Workflow `Context` owns
per-run execution state and events; they are not interchangeable.

## Workflow Design Checklist

- Use a returned list of events for finite fan-out.
- Use `Context.send_event` for dynamically discovered work.
- Use `Context.collect_events` to gather the expected fan-in set.
- Do not rely on completion order matching input order.
- Put serializable per-run values behind async `ctx.store` operations.
- Put clients, indexes, LLMs, and configuration in workflow resources.
- Use resource factories and validation for dependency injection.
- Do not attempt to checkpoint live resource objects as workflow state.
- Run `workflow.validate()` in tests or at startup.

```python
workflow = RagFlow(timeout=60)
workflow.validate()
```

Validation checks typed start/stop paths, events with no consumers, consumed
events with no producers, and dead-end execution paths.

## Upgrade Verification

After a migration or coordinated dependency upgrade, exercise all integration
boundaries the application uses: ingest new, unchanged, updated, and deleted
documents; persist and reload state; verify the remote collection and
embeddings; execute representative retrieval and agents; and stream, validate,
checkpoint, and resume important workflows.
