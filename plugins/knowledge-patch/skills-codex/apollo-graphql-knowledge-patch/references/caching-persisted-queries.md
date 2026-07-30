# Caching and Persisted Queries

## Response-cache configuration lifecycle

Router 2.8.0 introduces Redis-backed root-field and entity response caching
under `preview_response_cache`. It uses subgraph `Cache-Control` for TTL and
cache tags for targeted invalidation. Existing entity-cache configurations can
migrate by renaming their options.

Router 2.10.0 makes response caching generally available under
`response_cache`; production configuration should no longer use the preview
namespace.

In Router 2.9.0, the ineffective `ttl` nested under `redis` is removed. Put
fallback TTL on the relevant `preview_response_cache.subgraph` entry; after GA,
use the corresponding `response_cache` path:

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

Router 2.12.0 clarifies that a configured TTL applies only when a subgraph
omits `Cache-Control: max-age`. A multi-root-field subgraph response is stored
as one unit rather than one entry per root field.

Router 2.15.0 adds
`response_cache.include_cache_control_header_on_router_response`, defaulting to
`true`. Setting it to `false` suppresses client `Cache-Control` without
changing Redis storage, TTL, keys, or debugger behavior.

## Cache keys and identity

### Entity-key evolution

Router 2.1.0 separates entity key fields from representation-variable values
in cache keys, fixing `@requires`-related failures. This changes hashing for
distributed query-plan caching, so expect regeneration during upgrade.

Router 2.1.3 fixes entity caching for types with multiple `@key` directives
whose fields differ.

Router 2.6.0 changes entity-cache keys again while preventing storage of an
already expired response whose `Age` exceeds `max-age`. Plan another cold-cache
period.

### Nullable and interface keys

Router 2.10.0 treats Federation interface objects as entities for response
caching, allowing their representations to serve as keys.

Router 2.11.0 accepts nullable `@key` fields. Keep identities simple and avoid
a representation where `null` is ambiguous.

Router 2.13.0 additionally accepts a missing nullable key field and a nullable
list item whose value is `null`.

### Schema and per-subgraph identity

Router 2.12.0 includes schema changes in cache identity, so old entries stop
receiving hits rather than serving stale-schema data.

Router 2.9.0 lets `apollo::response_cache::key` contain a `subgraphs` map.
A named subgraph entry replaces rather than merges with `all`; repeat common
data inside the named entry.

```json
{
  "all": 1,
  "subgraphs": {
    "products": { "locale": "be" }
  }
}
```

Router 2.10.0 permits Rhai and coprocessors to customize identity at the
subgraph stage, for example by copying a request header to a `private_id`
context value.

## Cache-Control interpretation

Router 2.6.0 normalizes the client cache header for a single uncached entity
fetch using the same algorithm as other fetches. It emits `max-age`, not
`s-maxage`, instead of forwarding the subgraph header unchanged.

Router 2.13.0 makes a cached response containing GraphQL errors emit
`Cache-Control: no-store`, preventing intermediary storage.

The response and entity caches distinguish directives in 2.13.0:

- `no-store` may serve an existing entry but prevents a new store.
- `no-cache` prevents serving without revalidation but still permits storage.
- The Router does not implement the required revalidation.

Router 2.16.0 expands parsing and precedence:

- Numeric `stale-if-error` is accepted.
- `s-maxage` remains distinct from `max-age`.
- An extension-only header is `no-store`.
- Field-qualified `no-cache` is accepted.
- Future-dated entries expire.
- `private` can suppress `public`.
- Older Redis entries with boolean stale directives remain readable during a
  rolling upgrade.

## Invalidation

Router 2.10.0 starts the invalidation endpoint when invalidation is enabled
globally or for any individual subgraph. It is not necessary to enable
`response_cache.subgraph.all.invalidation.enabled` when only named subgraphs
accept invalidation.

Router 2.11.0 surfaces invalidation failures instead of letting them remain
silent, which can increase
`apollo.router.operations.response_cache.invalidation.error`. It also rejects
unknown request payload fields with HTTP 400.

Router 2.16.0 allows each subgraph to disable `subgraph`, `type`, or `cache_tag`
invalidation indexes. All default to enabled. Disabling an index avoids its
Redis writes, and a request using that invalidation kind returns HTTP 400.

```yaml
response_cache:
  enabled: true
  subgraph:
    all:
      enabled: true
      invalidation:
        enabled: true
        indexes:
          subgraph: false
          type: false
```

Re-enabling an index does not backfill entries created while it was off. Flush
the affected Redis namespace before re-enabling when existing entries must
immediately participate.

## Redis operation

Router 2.6.0 exposes stable query-plan cache metrics:

- `apollo.router.cache.redis.connections`
- `apollo.router.cache.redis.command_queue_length`
- `apollo.router.cache.redis.commands_executed`
- `apollo.router.cache.redis.redelivery_count`
- `apollo.router.cache.redis.errors`

Experimental metrics report average network latency, command latency, request
size, and response size:

```text
experimental.apollo.router.cache.redis.network_latency_avg
experimental.apollo.router.cache.redis.latency_avg
experimental.apollo.router.cache.redis.request_size_avg
experimental.apollo.router.cache.redis.response_size_avg
```

`metrics_interval`, default one second, controls collection frequency.

Router 2.8.0 replaces `apollo.router.cache.redis.connections` with
`apollo.router.cache.redis.clients`; it counts clients rather than underlying
connections and removes the `kind` attribute.

Router 2.9.0 standardizes Tokio and Redis response-cache timeout metrics on the
code `timeout` across `apollo.router.operations.response_cache.*.error`.

Router 2.10.0 sends read-only cache commands to Redis replicas in a cluster for
both query-plan and response caches.

Router 2.16.0 connects to replicas eagerly, avoiding read failures, backend
fallthrough, and CPU spikes from lazy round-robin routing with an even number
of replicas.

## Persisted-query manifests and safelists

### Local manifests

Router 2.1.0 adds `persisted_queries.hot_reload: true` for configured local
manifest files. This is independent of the Router's `--hot-reload` flag.

```yaml
persisted_queries:
  enabled: true
  local_manifests:
    - ./manifest.json
  hot_reload: true
```

Router 2.16.0 deprecates
`persisted_queries.experimental_local_manifests`. Rename it to the
behavior-equivalent `local_manifests`; the old key is scheduled for Router 3
removal.

### Usage and request context

Router 2.2.0 reports persisted-query usage keyed by persisted-query ID.
Router 2.7.0 also counts safelisted operations submitted by body, rather than
only ID-based requests.

Router 2.13.0 stores the resolved persisted-query ID in request context, where
Rhai can read it.

### Rejection and logs

Router 2.3.0 adds `enforcement_skipped` to unknown-operation safelist logs:
`false` means an external operation was rejected, while `true` means an
internal operation intentionally bypassed enforcement.

With manifest safelisting enabled and APQ disabled, Router 2.4.0 includes
`extensions.operation_name` on `PERSISTED_QUERY_NOT_IN_LIST` when the request
supplied an operation name.
