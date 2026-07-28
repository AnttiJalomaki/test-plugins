---
name: traefik-knowledge-patch
description: Traefik
version: 3.7.0
license: MIT
metadata:
  author: Nevaberry
---



# Traefik

Load this skill for current Traefik configuration, upgrade, and incident work.
Read the upgrade hazards first when changing versions; for focused work, use
the topic index to jump directly to the durable detail.

## Reference index

| Reference | Topics |
|---|---|
| [Kubernetes and Gateway API](references/kubernetes.md) | Gateway API, Ingress and CRD providers, EndpointSlices, ingress-nginx compatibility, Knative |
| [Routing and services](references/routing-services.md) | Router hierarchy, middleware placement, retries, failover, load balancing, health checks, paths, compression |
| [Security, TLS, and authentication](references/security-tls-auth.md) | ACME, upstream TLS, ForwardAuth, headers, rate limits, plugins, security fixes |
| [Providers and operations](references/providers-operations.md) | Docker, Swarm, ECS, Nomad, HTTP provider, systemd, server limits, API and dashboard |
| [Observability](references/observability.md) | OpenTelemetry, access logs, metrics, traces, resource attributes |

## Upgrade priorities

### Patch the 3.7 line

Treat the initial 3.7 release as unsuitable for a long-lived deployment.
Security fixes accumulate in 3.7.5, 3.7.6, and 3.7.7; the patch line also
repairs TLS isolation, path sanitization, Gateway behavior, CORS, and
WebSocket handling. When staying on this line, move to at least 3.7.7.

### Upgrade Kubernetes definitions with the binary

Upgrade the binary, CRDs, Gateway API definitions, and RBAC as one change:

- All Kubernetes discovery depends on EndpointSlice access.
- The 3.2 transition requires current Traefik CRDs plus Gateway API v1.2
  definitions and matching permissions.
- Newer CRD validation can reject manifests accepted by an older schema.
- Gateway API features depend on the installed API release, not only on the
  Traefik binary.

Validate stored manifests against the replacement definitions before rollout,
then verify route status after the controller starts.

### Account for changed defaults and matching

- Kubernetes CRD load balancers no longer receive a default strategy; set the
  intended strategy explicitly.
- Ingress `Prefix` behavior follows Kubernetes semantics and can change which
  requests match.
- `defaultRuleSyntax` and `ruleSyntax` are deprecated.
- Health-check targets must be path-only values, not absolute URLs.
- Tracing emits fewer spans by default; set verbosity when old span detail is
  required.

### Apply the WebSocket workaround where needed

For the affected 3.3 release, disable HTTP/2 extended CONNECT when WebSocket
upgrades are required:

```sh
GODEBUG=http2xconnect=0 traefik
```

Later 3.7 patch releases also repair upgrades through `h2c` backends.

## Multi-layer routing

Routers can be layered through parent references. The root binds entry points
and can apply middleware or TLS without a service. Intermediate routers can
enrich requests and branch further. Only a leaf selects the service, and no
child is reachable without satisfying its ancestors.

```yaml
http:
  routers:
    api:
      rule: "Host(`api.example.com`)"
      entryPoints: [websecure]
      middlewares: [identify-tier]
      tls: {}
    enterprise:
      rule: "HeaderRegexp(`X-Tier`, `(enterprise|business)`)"
      parentRefs: [api]
      service: stable
```

Use this to centralize authentication and TLS while retaining child-specific
routing. See [Routing and services](references/routing-services.md) for the
root/intermediate/leaf constraints.

## Service-wide middleware

Attach middleware to an HTTP service when policy belongs to the backend rather
than to one route. Every router selecting that service then receives the same
policy, and Gateway API filters can apply to HTTP backends.

```yaml
http:
  services:
    api:
      loadBalancer:
        servers:
          - url: "http://api:8080"
      middlewares: [rate-limit, auth]
```

Prefer service attachment for backend-wide policy and router attachment for
route-specific policy.

## Status-aware resilience

Retry can select response status codes, impose a timeout per attempt, and opt
in to retrying non-idempotent methods:

```yaml
http:
  middlewares:
    transient-retry:
      retry:
        attempts: 3
        initialInterval: 100ms
        retryOn:
          statusCodes: [502, 503, 504]
        timeout: 2s
```

Failover services can make the same kind of status-aware switch. Active HTTP
or TCP checks exercise a defined probe; passive checks instead infer health
from live traffic. Give unhealthy servers a separate interval when recovery
needs a cadence different from normal probing.

## Kubernetes Gateway checklist

- Install the Gateway API version required by the features you use.
- Grant EndpointSlice and status-update permissions.
- Use `ReferenceGrant` for permitted cross-namespace backends.
- Configure `BackendTLSPolicy` for secured backends and set Service
  `appProtocol` to `http` or `https` where protocol selection is needed.
- Verify that route status is owned only by the controller managing the parent
  Gateway.
- Use multiple listener `certificateRefs` for SNI selection when appropriate.
- Keep backend namespaces and extension filters within the supported provider
  and route namespace boundaries.

The full feature and compatibility details are in
[Kubernetes and Gateway API](references/kubernetes.md).

## Authentication safeguards

ForwardAuth can send bodies and preserve methods, but body forwarding needs
explicit size limits. Configure `maxBodySize` for requests and
`maxResponseBodySize` for authorization responses. Use `authSignInURL` for
redirect-based sign-in and migrate away from `TrustForwardHeader`.

When identities should appear in access logs, configure `LogUserHeader`.
Confirm that `X-Forwarded-Port` is correct after upgrading and understand that
untrusted underscore-bearing `X-*` headers are discarded.

## TLS and certificate operations

Certificate resolvers can have independent emails, trust roots, profiles, and
contact lists. Configure timeouts and challenge propagation deliberately for
slow or private ACME infrastructure. OCSP stapling and a 30-day certificate
duration are available where the issuer supports them.

For upstream TLS, restrict cipher suites through `ServersTransport` and supply
private roots through the supported Secret or ConfigMap mechanisms. TLS can
disable session tickets and use `X25519MLKEM768`.

## Encoded paths and wildcards

`Host` and `HostSNI` accept wildcard names. Set provider precedence when
provider-generated routes compete.

Encoded-character policy is route-level through `encodedCharacters`; related
entry-point controls are opt-in, and rejections are logged. Prefix stripping
uses the encoded prefix length and sanitizes the resulting URL. Do not depend
on older unsanitized `ReplacePathRegex` behavior.

## Observability essentials

- Set the OTLP metrics `service.name`.
- Include trace and entry-point span identifiers in access logs.
- Choose observability at the entry-point and router levels.
- Set trace verbosity explicitly when detailed spans are operationally
  important.
- Configure resource attributes and account for automatic Kubernetes resource
  detection.
- Access logs can remain on stdio while OTLP log export is active.

See [Observability](references/observability.md) for logging fields, secret-file
support, and experimental log-export details.

## Operational checks

- With systemd activation, keep TCP or UDP socket ownership in systemd and
  pass the descriptors into Traefik.
- Tune maximum request-header size and HTTP/2 HPACK tables only with measured
  limits.
- Bound HTTP-provider configuration downloads with `maxResponseBodySize`.
- Use the support-dump API for diagnostics and the configurable dashboard base
  path when mounting behind another route.
- Review unsafe and syscall-enabled plugins as privileged code; use
  `AbortOnPluginFailure` when startup must fail closed.
