# Routing, middleware, and services

## Hierarchical routers

Multi-layer HTTP routing arrived in 3.6.0. A child's `parentRefs` makes shared
middleware or TLS at the parent run before the child's own rule:

- A root attaches to entry points and does not select a service.
- An intermediate router may have children and enrich the request.
- A leaf must select a service.
- A child is not independently reachable; traffic must pass every parent.

This supports flows where ForwardAuth writes a response header and a child rule
then selects a backend from that header.

```yaml
http:
  routers:
    root:
      rule: "Host(`api.example.com`) && PathPrefix(`/`)"
      entryPoints: [websecure]
      middlewares: [auth-with-tier]
      tls: {}
    premium:
      rule: "HeaderRegexp(`X-Customer-Tier`, `(enterprise|business)`)"
      parentRefs: [root]
      service: stable
```

## Service-level middleware

Since 3.7.0, an HTTP service can name middleware directly. Every router that
uses the service receives the middleware, and Gateway API filters can be
applied to HTTP backends:

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

Do not duplicate service-wide policy on each router unless rule-local ordering
requires it.

## Retry and failover

The 3.7.0 Retry middleware can retry selected HTTP response statuses, set a
timeout per attempt, and opt in to non-idempotent methods:

```yaml
http:
  middlewares:
    retry-transient:
      retry:
        attempts: 3
        initialInterval: 100ms
        retryOn:
          statusCodes: [502, 503, 504]
        timeout: 2s
```

Treat non-idempotent retries as an application-level decision; a timeout does
not prove that an upstream operation had no effect.

Failover services can switch based on response status in 3.7.0.
`TraefikService` expresses this directly:

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

## Health checks

An HTTP health check can use a distinct interval while the server is unhealthy
(3.5.0). This separates normal probing cost from recovery detection.

Traefik 3.6.0 adds native TCP checks for non-HTTP backends and passive checks
that infer health from real requests. Choose active probes to test a known
contract and passive checks when synthetic traffic is undesirable.

Kubernetes CRD `ExternalName` Services support checks from 3.1.0. In 3.7.0,
health-check path validation requires a path-only value rather than an
absolute URL.

## Load-balancing strategies

The `p2c` strategy became an available server load-balancer choice in 3.4.0.
`Least Time`, added in 3.6.0 for file and Kubernetes CRD configuration, selects
servers with the lowest observed response time. `HighestRandomWeight` adds a
probabilistic weighted choice and is available through Kubernetes CRDs.

The Kubernetes CRD no longer supplies a default strategy as of 3.4.0. Make the
selection explicit in manifests that rely on a particular algorithm.

`NativeLB` in the Gateway provider delegates balancing to the Kubernetes
Service (3.2.0).

## Sticky sessions

Sticky-cookie configuration gained a `path` in 3.3.0 and a `domain` in 3.4.0.
Scope both explicitly when the cookie should be limited to a URL subtree or
shared across a controlled host set.

Kubernetes serving endpoints are considered when selecting sticky backends
from 3.3.0.

## Mirroring and backend URLs

HTTP mirroring has `mirrorBody` from 3.2.0, allowing each mirror service to
choose whether request bodies are copied. In 3.6.21, mirroring correctly
handles an empty body whose length is unknown.

HTTP services can preserve a configured backend server path when proxying
(3.2.0). Label-based providers can configure complete backend server URLs from
Docker, ECS, Swarm, Consul Catalog, and Nomad labels starting in 3.4.0.

## Host, path, and rule behavior

`Host` and `HostSNI` accept wildcard names such as `*.example.com` in 3.7.0.
When multiple providers produce competing routes, configure provider
precedence instead of relying on incidental order.

The `encodedCharacters` middleware adds route-level encoded-character policy
in 3.7.0. Related entry-point options are opt-in, and rejected requests appear
in access logs. Prefix stripping now measures the encoded prefix and sanitizes
the resulting URL. Release 3.7.7 also sanitizes paths emitted by
`ReplacePathRegex`.

Ingress `Prefix` matching follows Kubernetes semantics from 3.5.0; retest
routes that relied on the former behavior. The `defaultRuleSyntax` and
`ruleSyntax` settings are deprecated as of 3.4.0.

## Header, error, and compression middleware

The Headers middleware can emit report-only CSP via
`Content-Security-Policy-Report-Only` (3.1.0).

The Compress middleware can negotiate Zstandard when a client advertises
`zstd` (3.1.0). Its `encodings` option, added in 3.2.0, restricts which formats
may be selected.

The Errors middleware can rewrite the response status while serving an error
page from 3.4.0. In 3.7.0, `errorRequestHeaders` chooses which original request
headers are sent to the error service.

Gateway response-header filters can add, set, or remove response headers from
3.2.0. Patched 3.7 releases also allow Gateway header modifiers to change the
`Host` header.

Later 3.7 fixes ensure redirects preserve the incoming scheme when none is
configured and return the chosen status. CORS no longer emits an implicit
zero max-age and no longer combines credentialed requests with wildcard
origin.

## IP strategy and forwarded headers

`ipStrategy` can normalize IPv6 clients by subnet from 3.2.0.

In 3.7.0, a global setting can disable appending to `X-Forwarded-For`. The
server can remove incoming header names containing underscores, and
authentication middleware discards untrusted underscore-bearing `X-*`
headers.
