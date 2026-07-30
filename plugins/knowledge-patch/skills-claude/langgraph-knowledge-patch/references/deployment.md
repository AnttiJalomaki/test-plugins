# Deployment

## Graph loading

When a compiled graph is exported, Agent Server loads it once at container
startup and reuses it. A graph factory runs for every execution; reserve it
for per-run customization.

In both cases, the server injects the deployment's checkpointer and memory
store. Deployed graph code must not configure either one.

Agent Server can also host agents built with other frameworks. Adapt them for
deployment through the LangGraph Functional API or the
`deployments-wrap-sdk` package.

## Persistent and ephemeral backends

The storage roles are distinct:

- PostgreSQL always stores assistants, threads, runs, and cron jobs.
- Checkpoints default to PostgreSQL, but can use MongoDB or a custom backend.
- The long-term Store defaults to PostgreSQL but can be replaced.
- Redis is limited to ephemeral signaling, cancellation, and streaming
  pub/sub. It does not persist user or run data.

## Runtime layouts

Self-hosting supports three layouts:

1. Single-host mode is the default; the API server manages its task queue
   without separate workers.
2. Split mode independently scales API and worker pools and requires
   `queue.enabled: true`.
3. Distributed runtime separates graph orchestration from execution.

```yaml
queue:
  enabled: true
```

## Queue and concurrency boundaries

A worker leases a queued run from the durable database. The queue allows no
more than one executing run per thread.

Each worker processes at most `N_JOBS_PER_WORKER` jobs concurrently; its
default is `10`. This setting does not cap API request concurrency. A split
deployment must keep at least one queue worker listening.

## Threadless streaming

Pass `None` as the thread identifier for a threadless deployed run. The next
argument is the graph name from `langgraph.json`:

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

JavaScript LangGraph agents can run outside LangSmith on Next.js, SvelteKit,
Nuxt, Cloudflare Workers, or Deno Deploy while using the same Agent Streaming
Protocol.
