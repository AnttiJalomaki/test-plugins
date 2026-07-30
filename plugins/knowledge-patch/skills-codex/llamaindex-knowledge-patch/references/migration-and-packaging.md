# Migration and Packaging

## Removed Core Abstractions

Core v0.11 removes the deprecated `ServiceContext` and `LLMPredictor`
abstractions. Use `Settings` for process defaults:

```python
from llama_index.core import Settings

Settings.llm = llm
Settings.embed_model = embed_model
```

Pass LLMs, embedding implementations, and transformations directly when
configurations coexist. Direct injection avoids one process-wide setting
leaking across libraries, tests, tenants, or independently configured indexes.

The same transition moves core officially to Pydantic v2. Audit code that
subclasses or serializes framework objects, including:

- custom readers and nodes;
- output parsers and tools;
- LLM and embedding integrations;
- validators and model configuration;
- stored payloads and other serialized state;
- dependencies that still assume Pydantic v1 behavior.

Changing only `ServiceContext` construction can leave validators, schemas, and
serialization paths broken.

## Coordinated Dependency Upgrades

The v0.12 package generation bumps every `llama-index-*` distribution, but the
distributions do not all become version `0.12.0`. Integration packages retain
their own versions and declare constraints against core. Resolve a coherent
environment instead of upgrading only `llama-index-core` or pinning the entire
family to a shared number.

Inspect installed distributions when debugging an environment:

```python
from importlib.metadata import version

core_version = version("llama-index-core")
starter_version = version("llama-index")
```

A safe dependency-upgrade loop is:

1. state the application's Python version and direct requirements;
2. resolve every installed `llama-index-*` distribution together;
3. inspect the resolver's chosen integration versions and core constraints;
4. run ingestion, retrieval, agent, and workflow integration tests;
5. preserve the successful lockfile.

Do not assume interpreter support from the core release number alone. The
v0.12 package generation drops Python 3.8, while selected later integration
releases can impose a higher floor. Follow the metadata of the exact
distributions selected by the resolver.

## Agent API Migration

Current workflow agents are imported from:

```python
from llama_index.core.agent.workflow import (
    AgentWorkflow,
    FunctionAgent,
    ReActAgent,
)
```

Legacy provider-specific agent classes and runner/worker examples do not
describe the current execution contract. `run()` starts asynchronous workflow
execution and returns a streamable, awaitable handler.

Current memory guidance uses `Memory`. `ChatMemoryBuffer`,
`ChatSummaryMemoryBuffer`, and `VectorMemory` are deprecated. Create a
`Memory` instance explicitly and pass it to the agent run.

## Workflow Migration

A `QueryPipeline` DAG cannot be migrated by renaming its class. Redesign it as
a workflow with:

- typed Pydantic event classes;
- asynchronous `@step` methods;
- explicit event branches and loops;
- `Context` state access;
- streaming through the run handler;
- checkpoint-aware durable execution where required.

Core applications retain the `llama_index.core.workflow` import surface.
Standalone workflow applications can install `llama-index-workflows` and
import through `workflows`.

## Ingestion Migration

`VectorStoreIndex.from_documents` remains a useful convenience. Move
maintained ingestion flows to `IngestionPipeline` when they need
transformation caching, document update strategies, or direct vector-store
insertion.

An embedding change is a data migration concern. Rebuild or explicitly verify
persisted vector indexes after changing the embedding implementation or its
configuration. Successful deserialization does not prove vector
compatibility.

Upgrade verification should cover the full paths the application uses:
ingestion, retrieval, agent tools and memory, workflow streaming, validation,
checkpointing, and resource construction.

This guidance originates from the included
`llamaindex-v0.11-v0.12-api` compatibility batch.
