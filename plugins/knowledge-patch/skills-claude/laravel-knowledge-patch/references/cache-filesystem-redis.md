# Cache, Filesystems, Redis, Sessions, and Locks

## Cache stores and expiration

- **Cache flush events** `[2025-03]`: listen for `CacheFlushed` when application
  behavior must react to a store flush.
- **Memoized stores** `[2025-04]`: `Cache::memo()` wraps the default or named
  store with request-local memoization.
- **Flexible memoized reads** `[2025-05]`: memoized stores support
  `flexible()`, combining local memoization with stale-while-revalidate.
- **Always-deferred refreshes** `[2025-05]`: pass `alwaysDefer: true` to
  `Cache::flexible()` to defer regeneration even where the current context
  would otherwise refresh immediately.
- **Failover stores** `[2025-10]`: configure failover cache stores for fallback
  when a primary store fails; emitted cache events identify the underlying
  store that handled the operation.
- **Single failover notifications** `[2026-01]`: `CacheFailedOver` and
  `QueueFailedOver` fire only for the first failure in a failover attempt, not
  once per failed backend.
- **Lock-pruning control** `[2025-12]`: database-backed locks can disable their
  probabilistic pruning lottery when pruning is managed separately.
- **Typed cache retrieval** `[2026-02]`: cache repositories provide typed
  getters for enforcing the expected cached value type.
- **Driver-independent funnels** `[2026-02]`: `Cache::funnel()` provides
  concurrency limiting with any cache driver.
- **Wider database expirations** `[2026-03-laravel-12]`: custom database-cache
  migrations should use a big-integer expiration column.
- **Laravel 13 cache TTL contract** `[13.0-upgrade]`: custom `Store` and
  `Repository` implementations must implement `touch($key, $seconds)`.
- **Restricted unserialization** `[13.0-upgrade]`: the default
  `cache.serializable_classes` is `false`; allow-list cached object classes or
  use non-object payloads.
- **Memoized cache injection** `[2026-06]`: the `Cache` contextual attribute can
  use memoization for repeated reads in one execution.
- **Refreshable locks** `[2026-06]`: refresh a cache lock's expiration during
  long-running work instead of releasing and reacquiring it.

## Redis behavior

- **Predis 3 support** `[2025-05]`: Laravel supports `predis/predis` 3.x.
- **Redis cluster broadcasting** `[2025-08]`: broadcasting works with clustered
  Redis connections; a separate non-cluster connection is unnecessary.
- **PhpRedis number packing** `[2025-09]`: configure the client
  pack-ignore-numbers option so packing preserves numeric values as intended.
- **Atomic tagged flushes** `[2025-09]`: Redis cache-tag flushes are atomic and
  accept custom Redis connections.
- **Predis cluster keys** `[2025-11]`: `PredisClusterConnection::keys()` supports
  pattern-based key lookup.
- **Command failure listeners** `[2026-01]`: Redis connections expose
  `listenForFailures()` and dispatch `CommandFailed`.
- **PhpRedis keepalive** `[2026-03-laravel-12]`: configure `tcp_keepalive` on
  PhpRedis connections.
- **Cacheable Predis retries** `[2026-07]`: Predis retry configuration accepts
  scalar values and therefore works with `config:cache`.

## Sessions and password reset storage

- **Password reset expiry units** `[12.0-upgrade]`: custom
  `DatabaseTokenRepository` construction must pass expiry seconds, not minutes.
- **Cache reset prefix removed** `[2025-07]`: remove the `prefix` option from
  cache-backed password broker configuration.
- **Generic cache session driver** `[2025-09]`: select `SESSION_DRIVER=cache`
  and use `SESSION_STORE` to choose the cache store.
- **Laravel 13 identifier defaults** `[13.0-upgrade]`: fallback cache and Redis
  prefixes and session cookie names use hyphenated slugs with `-cache-`,
  `-database-`, and `-session`. Set `CACHE_PREFIX`, `REDIS_PREFIX`, and
  `SESSION_COOKIE` to preserve existing identifiers.
- **Redis session prefixes** `[2026-07]`: configure a session-specific Redis
  prefix to separate session keys from other Redis data.

## Filesystems and storage

- **Local fallback root** `[12.0-upgrade]`: without an explicit `local` disk,
  the root is `storage/app/private`, not `storage/app`.
- **Nested scoped disks** `[12.0.0]`: a scoped disk may use another scoped disk
  as its backing disk.
- **Scoped disk exceptions** `[2025-09]`: scoped disks pass `throw` to their
  parent disk, so configured write failures throw consistently.
- **Bounded traversal** `[2025-11]`: `Filesystem::files()` and
  `directories()` accept `depth:` for bounded recursion.
- **Local temporary uploads** `[2026-02]`: the local driver supports
  `temporaryUploadUrl()`.
- **Served URI uniqueness** `[13.0.0]`: every served filesystem disk must have
  a distinct URI; duplicate served URIs throw.
- **Encoded filesystem URLs** `[2026-05]`: generated URL paths are URL-encoded;
  local adapters preserve directory separators, including temporary URLs.
