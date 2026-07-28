# Kubernetes and Gateway API

Use this reference when installing Kubernetes API objects, configuring a
Kubernetes provider, or translating Ingress, Gateway API, or Knative resources.

## Install CRDs, RBAC, and discovery permissions

- The Kubernetes Gateway provider is a regular provider and uses Gateway API
  v1.1.0 (since 3.1.0); it no longer requires experimental-provider enablement.
- Every Kubernetes provider discovers backends through EndpointSlices (since
  3.1.0). Grant the Traefik controller RBAC access to EndpointSlices before an
  upgrade or no backends may be discovered.
- A 3.1-to-3.2 upgrade requires the updated Traefik CRDs. Gateway API users also
  need the v1.2 CRDs and matching RBAC (since 3.2.0).
- Updated Ingress CRDs enforce stronger CEL and HTTP-status regular-expression
  validation (since 3.4.0). Revalidate manifests because resources accepted by
  an older schema may be rejected after the CRD update.
- The CRD schema no longer supplies a default load-balancing strategy (since
  3.4.0). Set the desired strategy explicitly instead of relying on schema
  defaulting.
- Condition-only EndpointSlice updates trigger backend refreshes in patched
  3.6 releases (since 3.6.21), even when endpoint addresses do not change.

Apply API definitions before the resources that depend on them, and update
controller RBAC in the same rollout.

## Select Gateway API capabilities

### Match, filter, and status behavior

The Gateway provider adds these `HTTPRoute` capabilities in 3.1.0:

- HTTP-method and query-parameter matches;
- `RegularExpression` path matches;
- `HTTPURLRewrite`;
- redirect filters that set scheme or port;
- `ReferenceGrant` authorization for backends; and
- route status reporting, including invalid-route status.

It adds destination-port matching and `ResponseHeaderModifier` support in
3.2.0. The response-header filter can add, set, or remove response headers.
Patched 3.7 behavior also allows a Gateway API header modifier to change `Host`.

Redirect behavior in patched 3.7 releases retains the incoming scheme when a
filter does not specify one and emits the configured redirect status rather
than silently substituting another status.

### gRPC, TLS, and backend selection

- Gateway API v1.2.0 and `GRPCRoute` are supported since 3.2.0.
- `HTTPRoute` and `GRPCRoute` can choose backend protocols. Kubernetes Service
  `appProtocol` values for `http` and `https` are recognized, and
  `BackendTLSPolicy` can secure the connection to a backend (since 3.2.0).
- `NativeLB` lets Gateway API backends use Kubernetes-native Service load
  balancing (since 3.2.0).
- Traefik sets rule priority for competing `TLSRoute` rules (since 3.4.0).
- Gateway API v1.3 resources are supported since 3.5.0.
- Gateway API v1.4 is supported since 3.6.0. In that API release,
  `BackendTLSPolicy` and `SupportedFeatures` status reporting move from the
  Experimental channel to the Standard channel.
- Gateway API v1.5.1 is supported since 3.7.0. Listeners can contain multiple
  `certificateRefs` for SNI selection, and
  `BackendTLSPolicy.caCertificateRefs` may refer to Secrets that hold private
  CA bundles.

Patched 3.7 releases reject cross-provider `backendRefs.namespace` references.
They resolve backend `ExtensionRef` filters relative to the `HTTPRoute`
namespace rather than another resource's namespace.

### Capability and ownership status

Traefik publishes supported features into `GatewayClass` status (since 3.2.0),
so operators can discover controller capabilities. In clusters with more than
one Gateway controller, patched 3.6 releases ignore route `parentRefs` for
Gateways Traefik does not manage and update route-parent status only for managed
Gateways (since 3.6.21). This avoids taking status ownership from another
controller.

## Configure Ingress and CRD providers

### Backend discovery and publication

- Kubernetes Ingress and CRD providers can use internal node addresses for
  NodePort Services when external node addresses are absent or unsuitable
  (since 3.1.0).
- The CRD provider can health-check backends represented by Kubernetes
  `ExternalName` Services (since 3.1.0).
- An `IngressRoute` route can omit an explicit `kind` (since 3.3.0).
- Ingress and CRD providers recognize serving endpoints, including during
  sticky-session backend selection (since 3.3.0).
- The Ingress provider can publish ingress status for `ClusterIP` and
  `NodePort` Service types (since 3.4.0).
- CRD service TLS can load root CA certificates from ConfigMaps (since 3.4.0).
- Kubernetes Ingress can publish `ExternalName` Services through Traefik (since
  3.6.0).
- CRDs accept `ingressClassName` (since 3.7.0).

### Prefix matching

Kubernetes Ingress `Prefix` paths follow Kubernetes-documented matching
semantics (since 3.5.0). Audit overlapping and segment-boundary paths during an
upgrade; routes that depended on Traefik's earlier prefix interpretation may
match different requests.

## Migrate ingress-nginx resources

The ingress-nginx compatibility provider began as experimental in 3.5.0. It
became the first-class `kubernetesIngressNginx` provider in 3.7.0, so remove the
old experimental gate and configure it as a normal provider:

```yaml
providers:
  kubernetesIngressNginx:
    enabled: true
```

The provider recognizes more than 85 common annotations across authentication,
redirect and rewrite behavior, timeouts and buffering, affinity and canaries,
rate limiting, custom headers and errors, access logs, and per-Ingress entry
points. This is broad compatibility, not complete ingress-nginx emulation;
inventory and test every annotation used by a migration.

`configuration-snippet`, `server-snippet`, and `auth-snippet` have partial
support. Traefik parses them into structured, allowlisted directives and
rejects unsupported input. It never inserts arbitrary snippet text as raw NGINX
configuration.

Review these compatibility and safety controls (since 3.7.0):

- `AllowCrossNamespaceResources`;
- `GlobalAllowedResponseHeader`;
- `strictValidatePathType`; and
- `ipAllowListStrategy`.

## Run Knative workloads

The Knative provider discovers services, follows scaling events, and routes
traffic for Knative workloads (since 3.6.0). It remains experimental and can be
limited to selected namespaces:

```yaml
experimental:
  knative: true
providers:
  knative:
    namespaces:
      - serverless-apps
      - production
```

Knative v1.20 is supported since 3.7.0. Validate discovery and routing across a
scale-to-zero and scale-up cycle, not only while pods are already serving.

## Validate a Kubernetes change

1. Install the intended Traefik CRDs and Gateway API CRDs before dependent
   resources.
2. Verify EndpointSlice and status-update RBAC with the controller's service
   account.
3. Check `GatewayClass`, Gateway, and route status for accepted, supported, and
   invalid conditions.
4. Exercise `Prefix`, method, query, port, gRPC, TLS, redirect, rewrite, and
   response-header behavior used by the manifests.
5. Confirm that status is written only for Gateways managed by this controller.
