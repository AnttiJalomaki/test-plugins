# Observability, Auditing, and Web UI

## Cluster-wide operational visibility

### Telemetry

`/cluster/telemetry` returns information from all peers (since 1.17.0), so
operators no longer need to call each peer's `/telemetry` endpoint. It
includes cluster-wide activity such as leader elections, resharding, and shard
transfers.

### Optimization monitoring

`/collections/{collection_name}/optimizations` reports cluster-wide
optimization status and details of current and past operations (since 1.17.0).
The Web UI Optimizations tab shows the same domain as statuses, timelines, and
per-cycle task durations.

### Collection memory

Collection disk, RAM, and OS page-cache usage can be inspected by component,
including vectors, payload, and indexes (since 1.18.0). Values are aggregated
across the cluster and are available through an API and the collection detail
page's **Memory** tab.

### Per-collection API metrics

Pass `per_collection=true` to the metrics endpoint (since 1.18.0):

```http
GET /metrics?per_collection=true
```

This adds a `collection` label to `rest_responses_*` and `grpc_responses_*`,
providing per-collection request counts, failures, and response durations.

## Point search in the Web UI

The point-search interface can find points similar to a selected point, filter
by payload values, or locate a point by ID (since 1.17.0). Use it for
interactive inspection without treating it as a substitute for production
query monitoring.

## Audit logging

Qdrant can audit API operations that require authentication or authorization
(since 1.17.0). The resulting record covers protected actions.

### Cluster-wide audit queries

The audit-log query endpoint aggregates entries across every cluster node
(since 1.18.0). Entries include timestamp, API method, authentication type,
access result, and client information. Filter results by time range or any
field value rather than inspecting each node's log file.

### Correlation and tracing

Audit entries capture a caller-supplied tracing ID when the request contains
`x-request-id`, `x-tracing-id`, or `traceparent` (since 1.18.0). Supply one of
these headers to correlate an audited operation with client and distributed
tracing logs.

## Request-scoped inference credentials

External inference-provider API keys can be passed in a request header (since
1.17.0). This lets credentials accompany individual inference requests;
handle the header as a secret in clients, proxies, logs, and tracing systems.
