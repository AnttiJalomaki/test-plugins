# Apollo Router caching, traffic, subscriptions, and reloads

## Persisted queries and safelists

### Local manifests

`persisted_queries.hot_reload: true` watches configured local manifests
independently of the Router's `--hot-reload` flag (`2.1.0`).

```yaml
persisted_queries:
  enabled: true
  local_manifests:
    - ./persisted-query-manifest.json
  hot_reload: true
```

`persisted_queries.experimental_local_manifests` is deprecated in `2.16.0`; use
the behavior-equivalent `local_manifests` key shown above.

The Router reports persisted-query usage keyed by ID from `2.2.0`. Safelisted
operations sent by operation body also count from `2.7.0`.

With manifest safelisting enabled and APQ disabled, a
`PERSISTED_QUERY_NOT_IN_LIST` error includes `extensions.operation_name` when the
request supplied a name (`2.4.0`).

Router `2.13.0` puts the resolved persisted-query ID in request context, making it
available to Rhai.

Safelist unknown-operation logs include `enforcement_skipped` (`2.3.0`): `false`
means enforcement rejected an external operation; `true` means an allowed internal
bypass.

## Response caching

### GA namespace and storage

Router `2.8.0` introduced Redis root-field and entity-representation caching under
`preview_response_cache`, using subgraph `Cache-Control` TTLs and cache tags.
Router `2.10.0` makes it GA under `response_cache`; rename the namespace.

A configured TTL is only a fallback when the subgraph omits
`Cache-Control: max-age` (`2.12.0`). Schema changes generate new keys rather than
serving stale schema data, and multi-root subgraph responses are cached as a unit.

The ineffective `ttl` under `redis` was removed in `2.9.0`; put TTL on the
`preview_response_cache.subgraph` entry (or the equivalent GA
`response_cache.subgraph` entry):

```yaml
response_cache:
  enabled: true
  subgraph:
    all:
      enabled: true
      ttl: 10m
      redis:
        urls: ["redis://localhost:6379"]
```

Federation interface objects are entities for response caching from `2.10.0`.

### Entity keys and cache regeneration

Router `2.1.0` separates entity-key fields from representation variables in cache
keys, fixing directive cases such as `@requires` but changing the distributed
query-plan-cache hash. Router `2.1.3` fixes types with multiple distinct `@key`
directives.

Router `2.8.0` does not store already-expired entity responses whose `Age` exceeds
`max-age`; its key version changes, so plan for regeneration.

Nullable `@key` fields are accepted from `2.11.0`; avoid ambiguous null identity.
Router `2.13.0` additionally accepts a missing nullable field or a nullable-list
item set to `null`.

### Cache keys from context

`apollo::response_cache::key` can contain `all` plus a `subgraphs` map (`2.9.0`).
A subgraph entry replaces, rather than merges with, `all`; repeat shared values.

```json
{
  "all": 1,
  "subgraphs": {
    "my_subgraph": { "locale": "be" }
  }
}
```

Rhai and coprocessors can customize cache identity at the subgraph request stage
from `2.10.0`, for example by copying a request header to a `private_id` context
key.

### Client-facing `Cache-Control`

A single uncached entity fetch no longer forwards its header verbatim in `2.6.0`;
it follows the common algorithm, emitting `max-age` but not `s-maxage`.

When a cacheable response contains GraphQL errors, Router `2.13.0` emits
`Cache-Control: no-store` to prevent intermediary storage. In Router cache plugins:

- `no-store` may serve an existing entry but prevents a new store.
- `no-cache` prevents serving without revalidation but still permits storage; the
  Router itself does not perform revalidation.

`response_cache.include_cache_control_header_on_router_response` defaults to
`true` (`2.15.0`). Setting it false suppresses the client response header without
changing Redis storage, TTL, keys, or debugger behavior.

Router `2.16.0` accepts numeric `stale-if-error`, preserves `s-maxage` separately,
treats extension-only headers as `no-store`, permits field-qualified `no-cache`,
expires future-dated entries, and lets `private` suppress `public`. It reads older
Redis entries whose stale directives were booleans for rolling-upgrade
compatibility.

## Invalidation

From `2.10.0`, the invalidation listener starts when enabled globally or for any
individual subgraph; `subgraph.all.invalidation.enabled` is not required for
selective use. In the full path, this is
`response_cache.subgraph.all.invalidation.enabled`.

```yaml
response_cache:
  enabled: true
  invalidation:
    listen: 127.0.0.1:4000
    path: /invalidation
  subgraph:
    all:
      enabled: true
      redis:
        urls: ["redis://localhost:6379"]
    subgraphs:
      products:
        invalidation:
          enabled: true
```

Router `2.11.0` returns errors for invalidation failures rather than hiding them,
which can increase `apollo.router.operations.response_cache.invalidation.error`.
Payloads with unknown fields return HTTP 400.

Each subgraph can disable the `subgraph`, `type`, or `cache_tag` invalidation index
in `2.16.0`; all default on. Disabled index requests return 400 and skip Redis
writes. Re-enabling does not backfill older entries; flush the affected namespace
when old entries must participate immediately.

```yaml
response_cache:
  subgraph:
    all:
      invalidation:
        enabled: true
        indexes:
          subgraph: false
          type: false
```

## Redis clients

Query-plan Redis caches expose stable connection/client, queue, execution, retry,
and error metrics; detailed instruments are in `router-observability.md`.

Router `2.8.0` replaces `apollo.router.cache.redis.connections` with
`apollo.router.cache.redis.clients`. The gauge counts clients rather than
connections and removes `kind`.

With Redis cluster replicas, Router `2.10.0` sends read-only cache commands to
replicas. Router `2.16.0` connects to replicas eagerly, preventing read failure,
backend fallthrough, and CPU spikes with an even replica count.

Both Tokio and Redis response-cache timeouts use code `timeout` in
`apollo.router.operations.response_cache.*.error` from `2.9.0`.

## Traffic shaping and status codes

### Rate-limit semantics

Status behavior changed twice:

- Router `2.11.0` restored `429 Too Many Requests` / `TOO_MANY_REQUESTS` for
  enforced rate limits, classifying the response as throttling.
- Router `2.13.0` changed router/subgraph rate-limit or buffer-capacity exhaustion
  back to `503 Service Unavailable` / `SERVICE_UNAVAILABLE`, classifying it as
  overall service load.

Align retries and alerts with the exact installed minor.

### Connections and compression

`pool_idle_timeout` applies to subgraphs, Connector sources, and coprocessors
(`2.13.0`). It defaults to 15 seconds rather than the previous five; a null value
disables idle eviction.

```yaml
traffic_shaping:
  all:
    pool_idle_timeout: 30s
```

Traffic-shaping compression sets `content-encoding`; all subgraph requests
advertise `gzip`, `br`, or `deflate` through `accept-encoding` (`2.11.0`). These
headers are added after debug capture, so they are absent from the Connectors
Debugger.

`traffic_shaping.deduplicate_variables` is deprecated and ignored in `2.16.0`;
variable deduplication is always enabled.

### Known response sizes

From `2.9.0`, known-size GraphQL responses retain `Content-Length` rather than
switching to `transfer-encoding: chunked`, and size hints survive both Router and
subgraph paths.

## Subscriptions

### Deduplication identity

`subscription.deduplication.ignored_headers` allows differences in irrelevant
headers without splitting otherwise identical subgraph subscriptions (`2.3.0`).

Decoded JWT claims participate independently in subscription identity. In
`2.15.0`, `ignore_auth_context: true` can share only genuinely non-personalized
streams. Defaults and overrides may be set per subgraph.

```yaml
subscription:
  deduplication:
    all:
      enabled: true
    subgraphs:
      stocks:
        ignore_auth_context: true
```

### Protocol and lifecycle

Router `2.7.0` accepts `graphql-transport-ws` `connection_error` messages with a
payload but no `id`, forwarding underlying errors.

The Router injects trace headers into the initial subgraph WebSocket upgrade
request (`2.11.0`); individual messages cannot add propagation headers.

Self-hosted subscriptions are available on every GraphOS plan from `2.11.0`, but
remain licensed and require GraphOS connection with API key and graph ref.

`subscription.max_lifetime` (`2.15.0`) closes an overlong subscription with
`SUBSCRIPTION_MAX_LIFETIME_EXCEEDED`; unset remains unlimited.

```yaml
subscription:
  enabled: true
  max_lifetime: 10m
```

AWS REST API Gateway can front multipart subscriptions now that response streaming
is available (`2.13.0`); configure the gateway response transfer mode for
streaming.

## Reloading and deployment artifacts

Tag-based OCI references, including generated variant and custom tags, are polled
and reloaded when their target changes (`2.11.0`). This supports mutable promotion
such as `artifacts.apollographql.com/my-org/my-graph:prod`.

From `2.15.0`, a transient schema or related reload failure retries instead of
permanently serving the previous schema. `reload.max_retries` defaults to five;
`0` disables retries and null allows unlimited retries. `retry_delay` defaults to
ten seconds. A new trigger resets the budget.

```yaml
reload:
  max_retries: 5
  retry_delay: 10s
```

Warning-state licenses now enforce restricted-feature blocks (`2.11.0`); a
deployment using a blocked feature fails rather than continuing.

The Router Helm chart supports Deployment-level `deploymentAnnotations` separately
from `podAnnotations` (`2.7.0`). From `2.13.0`, `ServiceMonitor.metadata.name`
uses the `router.fullname` helper, honoring `nameOverride` and `fullnameOverride`;
a default release `my-release` changes to `my-release-router`.

Health-check endpoints can again be disabled from `2.3.0`.
