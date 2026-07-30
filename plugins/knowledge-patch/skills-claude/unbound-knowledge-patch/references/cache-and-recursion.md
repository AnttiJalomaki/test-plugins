# Cache and recursion

## Cache fill and failure caching

### Named forwarders

Since 1.21.0, a nonzero `target-fetch-policy` causes cache fill to fetch and
cache addresses for forward-host names. Negative AAAA cache entries also stop
the associated recursion correctly.

### Limit-triggered failures

Since 1.21.0, more resolver-limit failures cache the resulting `SERVFAIL` for
a short period. This reduces repeated work and means immediate retries can
observe the cached failure.

## TTL enforcement

### NSEC restored from cachedb

Since 1.22.0, NSEC records in messages restored from cachedb have their TTL
limited rather than retaining an excessive cached value.

### Prefetch through a CNAME

Since 1.22.0, prefetch TTL is limited after a short-TTL CNAME so prefetched
data cannot outlive the alias path.

### Zero TTL and expired replies

Since 1.24.0:

- Cached records expire when their TTL reaches 0.
- Upstream TTL-0 answers are not stored by cachedb.
- `serve-expired-reply-ttl` is capped by the record's original TTL.
- High-order-bit TTL values are decoded as positive per RFC 8767 section 4
  instead of being treated as zero.

### DNAME and synthesized CNAME

Since 1.25.0, cache TTL policy applies to DNAME and synthesized-CNAME data on
the wire path. A response synthesized from a TTL-0 DNAME can be reused from
cache for a one-second grace period to avoid repeated recursion, while the
client still receives TTL 0. Out-of-zone DNAME records are excluded from
CNAME synthesis.

## Serve-expired behavior

Since 1.23.0, `serve-expired-ttl` defaults to `86400` and
`serve-expired-client-timeout` defaults to `1800`, following RFC 8767.

Also since 1.23.0, not-yet-validated updates can refresh the cache again,
fixing a 1.22.0 regression that caused extra upstream traffic. Current
delegation and validation-recursion data can replace expired state. As a
consequence, less older DNSSEC-validated expired data may remain available
later.

## Redis cachedb

### Read-only replicas

Since 1.23.0, the Redis backend provides `redis-replica-*` options for using
read-only replicas.

### Reload independence

Since 1.23.0, a reload does not fail when Redis cannot connect or respond.
Unbound logs the error and checks expiration features when the server is
available.

### Case-insensitive keys

Since 1.23.0, cachedb hashing ignores case, so DNS names differing only by
letter case share the same Redis-backed records.

### Outage reconnect throttling

Since 1.24.0, the Redis backend detects a down server and throttles reconnect
attempts.

## EDNS Client Subnet

Since 1.24.0, failure to create an ECS subquery returns `SERVFAIL` instead of
`FORMERR`.

For `0.0.0.0/0`:

- Data without configured subnet treatment enters the global cache.
- Data with configured subnet treatment enters the subnet cache.

## External cachedb exclusions and validation

Since 1.25.0, `forward-no-cache` and `stub-no-cache` prevent both lookup and
storage in an external cachedb, including relevant ECS paths.

Expired bogus data is no longer returned as non-bogus, and cached
aggressive-negative replies carry the RA flag.

## NAT64 infrastructure cache

Since 1.24.0, NAT64-synthesized target addresses attach at the delegation
point, allowing the infrastructure cache to work correctly for NAT64.
