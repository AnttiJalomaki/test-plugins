# API migrations

Use this guide when upgrading a pre-workflow LlamaIndex application or changing
its installed package set. The version-dependent compatibility material comes
from batch `llamaindex-v0.11-v0.12-api`.

## Replace `ServiceContext` and `LLMPredictor`

Core 0.11 removes both deprecated abstractions. Choose configuration scope
explicitly:

- Use `Settings` for process defaults shared by the application.
- Pass LLM, embedding, and transformation objects directly when separate
  configurations coexist.

```python
from llama_index.core import Settings

Settings.llm = llm
Settings.embed_model = embed_model
```

The core transition also makes Pydantic v2 the supported foundation. Audit code
that depends on Pydantic behavior rather than assuming imports are the only
change. The audit includes custom readers, nodes, output parsers, tools, models,
validators, serialization, and integrations that depend on Pydantic v1.

## Resolve a coherent package environment

The 0.12 transition bumps every `llama-index-*` package, but integration
distributions retain independent versions and core constraints. Resolve the
package set as a unit and test that result. Avoid both of these shortcuts:

- Upgrading `llama-index-core` without resolving its integrations.
- Assigning version `0.12.0` to every LlamaIndex distribution.

Inspect the installed distributions when diagnosing a mixed environment:

```python
from importlib.metadata import version

core_version = version("llama-index-core")
starter_version = version("llama-index")
```

The release drops Python 3.8. Selected later packages can set an even higher
Python floor, so use each selected distribution's metadata as the authority.
Once the environment resolves, preserve its lockfile.

## Migrate agents to workflow execution

Current agent classes live under `llama_index.core.agent.workflow`:

```python
from llama_index.core.agent.workflow import (
    AgentWorkflow,
    FunctionAgent,
    ReActAgent,
)
```

They execute through asynchronous workflow handlers and events. Code based on
older `OpenAIAgent`, runner, or worker examples needs an execution-model rewrite,
not just a new import. Replace deprecated `ChatMemoryBuffer`,
`ChatSummaryMemoryBuffer`, and `VectorMemory` usage with explicit `Memory`.

## Redesign `QueryPipeline` DAGs

Workflows model control flow through:

- Typed Pydantic events.
- Asynchronous `@step` methods.
- Event branches and loops.
- Per-run `Context` state.
- Streaming and checkpointed durable execution.

These semantics are not a `QueryPipeline` rename. Map the old DAG's branches,
joins, state, and outputs onto steps and events deliberately.

For an application already using core, the stable workflow surface is
`llama_index.core.workflow`. A standalone workflow application can install
`llama-index-workflows` and import through `workflows`.

## Verify the upgrade end to end

`VectorStoreIndex.from_documents` remains useful for simple construction, but
make document updates explicit with `IngestionPipeline` when caching, update
strategy, or direct vector-store insertion matters.

Rebuild or verify persisted vector indexes whenever the embedding model or its
configuration changes. An upgrade test should cover ingestion, retrieval,
agent, and workflow integration paths rather than exercising only imports.
