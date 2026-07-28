# Routing, middleware, and services

Use this reference when composing router rules, middleware chains, backend
services, retries, failover, health checks, or load balancing.

## Compose router hierarchies

Routers can form a `parentRefs` hierarchy (since 3.6.0). A parent can apply
shared middleware or TLS configuration and enrich a request before a child
evaluates its rule.

```yaml
http:
  routers:
    api-parent:
      rule: "Host(`api.example.com`) && PathPrefix(`/`)"
      middlewares:
        - auth-with-tier
      entryPoints:
        - websecure
      tls: {}
    api-enterprise:
      rule: "HeaderRegexp(`X-Customer-Tier`, `(enterprise|business)`)"
      service: stable-backend
      parentRefs:
        - api-parent
  middlewares:
    auth-with-tier:
      forwardAuth:
        address: "http://auth-service:8080/validate"
        authResponseHeaders:
          - X-Customer-Tier
  services:
    stable-backend:
      loadBalancer:
        servers:
          - url: "http://api-v1-stable:8080"
```

Enforce these invariants:

- a root router attaches to entry points and has no service;
- an intermediate router may have children;
- a leaf router selects the service; and
- a child is reachable only by traversing its parent.

The ordering matters when a parent middleware adds a header used by a child's
rule, as in the example.

## Match hosts and choose provider precedence

`Host` and `HostSNI` accept wildcard host names such as `*.example.com` (since
3.7.0). Test both a matching subdomain and the bare parent domain; do not assume
the wildcard covers both.

Provider routing precedence is configurable (since 3.7.0). Set it explicitly
when equivalent routes can be produced by, for example, a Kubernetes provider
and another dynamic provider.

The `defaultRuleSyntax` and `ruleSyntax` options are deprecated (since 3.4.0).
Avoid using them to preserve legacy parsing; migrate rules to the current
syntax.

## Attach middleware to services

An HTTP service can have middleware attached directly (since 3.7.0). Every
router selecting that service receives the middleware, and Gateway API filters
can consequently apply to HTTP backends.

```yaml
http:
  services:
    api:
      loadBalancer:
        servers:
          - url: "http://api-backend:8080"
      middlewares:
        - rate-limit
        - auth
```

Use service-level attachment for a true backend invariant. Keep route-specific
policy on routers so unrelated consumers of the same service are not changed.

## Compress responses

The Compress middleware negotiates Zstandard when a client advertises `zstd`
(since 3.1.0). Its `encodings` option limits which compression formats may be
negotiated (since 3.2.0). Configure the list when backend, cache, or client
compatibility requires a narrower set, then test `Accept-Encoding` preference
and fallback behavior.

## Control encoded paths and rewrites

The `encodedCharacters` middleware supplies route-level policy for encoded
characters (since 3.7.0). Related entry-point policies are opt-in, and requests
they reject are recorded in access logs.

Prefix stripping now measures the encoded prefix length and sanitizes the
resulting URL. Patch 3.7.7 also sanitizes paths produced by
`ReplacePathRegex`. Exercise encoded separators, percent encodings, stripped
prefixes, and regex replacements at the same byte forms clients send.

## Mirror requests safely

HTTP mirroring exposes `mirrorBody` (since 3.2.0), allowing a mirror service to
decide whether request bodies are copied. Disable body mirroring when payloads
are sensitive, large, or unnecessary to the shadow workload.

Patched 3.6 releases correctly handle an empty mirrored request body whose
length is unknown (since 3.6.21). Include chunked or otherwise unknown-length
empty requests in mirroring regressions.

## Preserve backend URL paths

HTTP services can preserve the path configured on a backend server URL (since
3.2.0). Enable this behavior when a server URL contains a required base path;
test it together with strip-prefix and replace-path middleware to avoid an
unexpected double prefix or erased path.

## Rewrite error responses

The Errors middleware can rewrite the HTTP status while handling an error
response (since 3.4.0). In 3.7.0 it also gains `errorRequestHeaders`, which
selects the request headers forwarded to the error service. Keep identity and
credential headers out unless the error service explicitly needs and protects
them.

## Share rate-limit state

The RateLimit middleware can store state in Redis (since 3.4.0), allowing a
limit to be enforced across multiple Traefik instances. Validate Redis failure
behavior, key scope, and aggregate limits with more than one proxy replica.

## Configure sticky cookies

Sticky-session cookies accept a path (since 3.3.0) and a domain (since 3.4.0).
Use them to narrow or deliberately share cookie scope. Kubernetes Ingress and
CRD providers also recognize serving endpoints during sticky-session backend
selection (since 3.3.0).

Test cookie issuance, return routing, path boundaries, subdomains, and backend
changes rather than checking only the cookie attributes.

## Select a balancing strategy

- `p2c` is available as a service server load-balancing strategy (since 3.4.0).
- `Least Time` routes to servers with the lowest response times and is available
  in file-provider and Kubernetes CRD service configuration (since 3.6.0).
- `HighestRandomWeight` performs probabilistic weighted selection and is
  available through Kubernetes CRDs (since 3.6.0).

Strategy availability differs by provider. The Kubernetes CRD schema no longer
inserts a default strategy (since 3.4.0), so specify it when behavior must be
stable.

## Configure active and passive health

- A failed backend can use a distinct unhealthy probe interval (since 3.5.0).
- Native TCP health checks support non-HTTP services (since 3.6.0).
- Passive health checks infer health from real traffic instead of active probes
  (since 3.6.0).
- Kubernetes CRD services backed by `ExternalName` Services support health
  checks (since 3.1.0).
- Health-check paths must be path-only, not absolute URLs (since 3.7.0).

Choose active HTTP, active TCP, or passive observation according to the
backend's protocol and failure signal. Verify transitions into and out of the
unhealthy state at the configured cadence.

## Retry by response status

The Retry middleware can select response status codes, impose a per-attempt
timeout, and opt in to retrying non-idempotent methods (since 3.7.0):

```yaml
http:
  middlewares:
    smart-retry:
      retry:
        attempts: 3
        initialInterval: 100ms
        retryOn:
          statusCodes: [502, 503, 504]
        timeout: 2s
```

Only enable non-idempotent retries when the backend has a deduplication or
idempotency contract. Test timeout and status selection separately so slow
attempts are not mistaken for status-triggered retries.

## Fail over by response status

Failover services can switch to a fallback on selected response statuses, and
`TraefikService` CRDs can express that policy (since 3.7.0):

```yaml
apiVersion: traefik.io/v1alpha1
kind: TraefikService
metadata:
  name: api-failover
spec:
  failover:
    service: api-primary
    fallback: api-backup
    healthCheck: {}
    errors:
      status: ["500-504"]
```

Exercise healthy primary responses, each configured error range, an unhealthy
primary, and fallback failure. Retries and failover can multiply backend
attempts, so verify their combined behavior when both are present.

## Retest patched HTTP behavior

Patched 3.7 releases correct several externally visible behaviors:

- Gateway API header modifiers may change `Host`.
- A redirect with no explicit scheme retains the incoming scheme and uses the
  configured status.
- CORS no longer emits a default zero max-age and no longer combines a
  credentialed request with wildcard origin.
- WebSocket upgrades work with `h2c` backends.

Treat these as regression cases when updating within the 3.7 line, particularly
if clients or tests encoded the previous output.
