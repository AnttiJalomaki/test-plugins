# Deployment

Source batch: `platform-and-deployment`.

## Graph loading and server-owned persistence

When a compiled graph is exported, Agent Server loads it once when the
container starts and reuses it. A graph factory runs once per run; reserve it
for genuine per-run customization.

In both cases, Agent Server injects the deployment's checkpointer and memory
Store. Do not configure those objects in the graph code deployed to the
server.

## Adapting other agent implementations

An Agent Server graph does not need a native LangGraph implementation. Adapt
an agent built with another framework through the LangGraph Functional API or
the `deployments-wrap-sdk` package.

## Durable and ephemeral storage roles

Assistants, threads, runs, and cron jobs always persist in PostgreSQL.
Checkpoints default to PostgreSQL but may use MongoDB or a custom backend. The
long-term Store also defaults to PostgreSQL and may be replaced.

Redis is only for ephemeral signaling, cancellation, and streaming pub/sub. It
does not persist user or run data.

## Runtime layouts

Self-hosting supports three layouts:

- Single-host mode is the default; the API server manages the task queue
  without separate workers.
- Split mode independently scales API and worker pools. Enable it with
  `queue.enabled: true`.
- Distributed runtime separates graph orchestration from execution.

```yaml
queue:
  enabled: true
```

## Queue and concurrency boundaries

A worker leases a queued run from the durable database. The queue permits at
most one executing run for each thread.

Each worker runs up to `N_JOBS_PER_WORKER` jobs concurrently; the default is
`10`. This setting does not cap API request concurrency. A split deployment
must always have at least one queue worker listening.

## Threadless remote streaming

Pass `None` as the thread identifier to stream a threadless run. The next
argument is the deployed graph name configured in `langgraph.json`.

```python
from langgraph_sdk import get_sync_client

client = get_sync_client(url=deployment_url, api_key=langsmith_api_key)
for chunk in client.runs.stream(
    None,
    "agent",
    input={"messages": [{"role": "human", "content": "Hello"}]},
    stream_mode="updates",
):
    print(chunk.event, chunk.data)
```

## JavaScript deployment targets

JavaScript LangGraph agents can be deployed outside the managed platform on
Next.js, SvelteKit, Nuxt, Cloudflare Workers, or Deno Deploy while using the
same Agent Streaming Protocol.
