# Cache, resolution, and validation

## Named forwarders and glue

When `target-fetch-policy` is nonzero, cache fill fetches and stores addresses
for forward-host names (since 1.21.0). A negative AAAA cache entry now also
stops the related recursion correctly.

`harden-unverified-glue` governs missing-AAAA lookups initiated by cache fill
(since 1.22.0), making the hardening decision consistent on that path.

Unbound warns when stub-zone or forward-zone name servers are configured as
hostnames instead of IP addresses and resolving those hostnames would create a
circular dependency (since 1.24.0).

## Limit-triggered and negative caching

More resolver-limit failures are cached as short-lived `SERVFAIL` responses
(since 1.21.0). Immediate retries may therefore observe the cached failure
instead of starting the same work again.

`unbound-control flush_negative` reports the removed negative-cache data
correctly (since 1.23.0).

## TTL and alias handling

NSEC records restored from cachedb have their TTL capped instead of retaining
an excessive stored value (since 1.22.0). Prefetch TTL is also capped after a
CNAME with a short TTL so prefetched data cannot outlive the alias path.

The following expiry rules apply (since 1.24.0):

- Cached records expire on reaching TTL 0.
- Upstream TTL-0 answers are not stored by cachedb.
- `serve-expired-reply-ttl` is capped by the record's original TTL.
- TTL values with the high-order bit set are interpreted as positive according
  to RFC 8767 section 4 rather than forced to zero.

Cache TTL policy applies to DNAME records and synthesized CNAME data on the
wire path (since 1.25.0). A response synthesized from a TTL-0 DNAME can be
reused internally for a one-second grace period to avoid repeated recursion,
while clients still receive TTL 0. An out-of-zone DNAME is not used for CNAME
synthesis.

## Serve-expired behavior

The RFC 8767-aligned defaults are `serve-expired-ttl: 86400` and
`serve-expired-client-timeout: 1800` (since 1.23.0).

Cache refresh again accepts not-yet-validated updates, correcting behavior
introduced in 1.22.0 that generated extra upstream traffic (since 1.23.0).
Current delegation and validation-recursion data may replace expired state.
The tradeoff is that less older, DNSSEC-validated expired data may remain
available afterward.

## Redis and external cachedb

The Redis cachedb backend provides `redis-replica-*` options for read-only
replicas (since 1.23.0). Redis keys are hashed without regard to DNS-name
letter case, so differently cased spellings share records.

A reload no longer fails solely because Redis is unreachable or unresponsive
(since 1.23.0). Unbound logs the backend error and checks expiration features
when the server is available. Reconnect attempts are throttled after the
backend is detected as down (since 1.24.0).

`forward-no-cache` and `stub-no-cache` prevent both lookup and storage in an
external cachedb, including the relevant EDNS Client Subnet paths (since
1.25.0). Expired bogus data is not returned as non-bogus, and cached
aggressive-negative replies carry the RA flag.

## EDNS Client Subnet

The subnet cache is no longer inserted implicitly by a build configured with
subnet support (since 1.23.0). Enable it explicitly:

```conf
server:
    module-config: "subnetcache validator iterator"
```

Failure to create an ECS subquery returns `SERVFAIL`, not `FORMERR` (since
1.24.0). For `0.0.0.0/0`, data without configured subnet treatment enters the
global cache; data with configured subnet treatment enters the subnet cache.

## NAT64 and DNS64

NAT64-synthesized target addresses attach at the delegation point so infra
cache processing works correctly (since 1.24.0). The interaction between
`do-nat64` and `do-not-query-address` also applies consistently during retry
processing (since 1.25.0).

## DNSSEC validation

Alias-chain validation is stricter (since 1.25.0):

- YXDOMAIN is accepted only when accompanied by a DNAME.
- Signatures produced by revoked DNSKEYs are rejected.
- DNAME-to-CNAME and wildcard-CNAME chains receive stronger trust checks.

`iter-scrub-rrsig` caps the number of RRSIG records retained by iterator
scrubbing and defaults to `8` (since 1.25.0):

```conf
server:
    iter-scrub-rrsig: 8
```
