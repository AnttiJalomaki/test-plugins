# Kubernetes and Gateway API

## Gateway API lifecycle and routing

In 3.1.0 the Kubernetes Gateway provider left experimental status and adopted
Gateway API v1.1.0. Its HTTPRoute support includes method and query-parameter
matches, `RegularExpression` paths, `HTTPURLRewrite`, scheme- and port-aware
redirect filters, backend `ReferenceGrant` handling, and status for valid and
invalid routes.

With 3.2.0, install Gateway API v1.2.0 when using the corresponding provider
features. `GRPCRoute` is supported. HTTPRoute and GRPCRoute backends can select
protocols using the `http` and `https` Kubernetes Service `appProtocol` values,
and `BackendTLSPolicy` describes TLS-secured backends. HTTPRoute also supports
destination-port matches and `ResponseHeaderModifier`. `NativeLB` delegates
load balancing to the Kubernetes Service. Traefik advertises implemented
features in `GatewayClass` status.

Gateway API v1.3 resources are supported from 3.5.0.

Gateway API v1.4 is supported from 3.6.0. In that API release,
`BackendTLSPolicy` and supported-feature status graduate from the Experimental
channel to the Standard channel.

Gateway API v1.5.1 is supported in 3.7.0. A listener may provide multiple
`certificateRefs`, enabling SNI-based certificate selection.
`BackendTLSPolicy.caCertificateRefs` may point to Secrets that contain private
CA bundles. Patched 3.7 releases reject cross-provider
`backendRefs.namespace` references and resolve backend `ExtensionRef` filters
relative to the HTTPRoute namespace.

Later 3.7 patches also let header modifiers change `Host`; redirects retain the
incoming scheme when no scheme is configured and emit the configured status.

## Ownership and status refresh

As of 3.6.21, the provider ignores route `parentRefs` for Gateways it does not
manage. It updates route-parent status only for managed Gateways, preventing
controllers from claiming one another's status in a multi-controller cluster.

Traefik continues to report invalid routes and controller capabilities, so
status permissions are required in addition to read permissions.

## EndpointSlices and service discovery

All Kubernetes providers switched backend discovery to EndpointSlices in
3.1.0. Upgrade RBAC before the process: it must list, watch, and read the
EndpointSlice resources used by the providers.

In 3.6.21, discovery began reacting to condition-only EndpointSlice updates.
Backend health now refreshes when readiness or another endpoint condition
changes even if the address list is unchanged.

The Ingress and CRD providers can use node internal addresses for NodePort
Services when external node addresses are unavailable or inappropriate
(3.1.0). The CRD provider also supports health checks for `ExternalName`
Services.

Serving endpoints are recognized by the Ingress and CRD providers, including
when selecting a sticky-session backend (3.3.0). The Ingress provider can
publish `ClusterIP` and `NodePort` Service status (3.4.0), and it can publish
`ExternalName` Services through Traefik (3.6.0).

## CRD upgrade behavior

The 3.1-to-3.2 Kubernetes upgrade requires the updated Traefik CRDs; Gateway
installations additionally require the v1.2 Gateway API CRDs and matching RBAC
(3.2.0).

CRDs in 3.4.0 apply stronger CEL validation and stricter regular-expression
validation for HTTP status codes. Validate manifests before swapping the CRDs:
an object admitted by an older schema may be rejected by the new one.

The CRD schema no longer supplies a default service load-balancing strategy in
3.4.0. Set the desired strategy in every manifest that depends on one.
Kubernetes CRD service TLS can load additional root CAs from ConfigMaps.

`IngressRoute` definitions may omit the explicit route `kind` from 3.3.0.
Kubernetes CRDs add `ingressClassName` in 3.7.0.

For Gateway TLS routing, Traefik assigns a priority to `TLSRoute` rules
(3.4.0), which determines precedence when multiple TLS routes compete.

## Kubernetes Ingress semantics

Starting in 3.5.0, Ingress `Prefix` paths match according to the Kubernetes
documented semantics. Audit overlapping paths and trailing segments when
upgrading from the earlier behavior.

Health-check paths are validated in 3.7.0 and must be URL paths, not complete
absolute URLs.

Patched 3.7 releases isolate TLS options for the same host on different entry
points, perform SNI checks for routers without host rules, and select
certificates deterministically when multiple certificates share a SAN.

## Ingress NGINX compatibility provider

The provider arrived experimentally in 3.5.0. It consumes existing
ingress-nginx resources, but that first implementation covered common use
cases and essential annotations rather than complete compatibility. Inventory
the exact annotations in a migration and test each one.

In 3.7.0, `kubernetesIngressNginx` became a first-class provider and no longer
requires the experimental flag:

```yaml
providers:
  kubernetesIngressNginx:
    enabled: true
```

It supports more than 85 common annotations across authentication, redirects
and rewrites, timeouts and buffering, affinity and canaries, rate limits,
custom headers and errors, access logs, and per-Ingress entry points.

`configuration-snippet`, `server-snippet`, and `auth-snippet` are only
partially compatible. Traefik parses allowlisted structured directives and
rejects unsupported text instead of injecting raw NGINX configuration.
Review `AllowCrossNamespaceResources`, `GlobalAllowedResponseHeader`,
`strictValidatePathType`, and `ipAllowListStrategy` when matching an existing
controller's behavior and security boundaries.

Access logs can include Kubernetes Ingress fields in 3.7.0.

## Knative

The experimental Knative provider was added in 3.6.0. It discovers Services,
tracks scaling events, and routes Knative workloads. Enable it and bound its
watch scope:

```yaml
experimental:
  knative: true
providers:
  knative:
    namespaces: [serverless-apps, production]
```

Knative v1.20 compatibility is available in 3.7.0.
