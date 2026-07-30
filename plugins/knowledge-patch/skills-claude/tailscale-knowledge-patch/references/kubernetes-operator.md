# Kubernetes Operator

## Build highly available application proxies

Operator-managed Ingresses and Tailscale Kubernetes Services can use a
`ProxyGroup` for multiple active proxy replicas and multiplex multiple
applications across those replicas (1.84.0). Both resource types can expose
applications across clusters. The Operator watches `EndpointSlice` objects
cluster-wide and can fail over when a cluster has no healthy backends.

Egress `ProxyGroup` replica restarts no longer interrupt cluster workloads
that access tailnet targets (1.80.0). Direct connections to `ProxyGroup` Pods
also improve when container image v1.86.2 advertises external node IP addresses
as static endpoints (1.86.0).

Operator-created proxy `ServiceMonitor` resources accept user-specified labels
(1.80.0). Changed proxy `tailscaled` configuration reloads dynamically;
hostname changes and similar values can take up to one minute to propagate.

## Expose and validate Kubernetes resources

- An unset path on an Operator-managed Ingress defaults to `/` (1.84.0).
- Configure Let's Encrypt staging certificates through the `ProxyClass` APIs
  while testing Ingress TLS, avoiding production rate limits (1.82.0).
- Apply `tailscale.com/http-redirect` to an Ingress to enable an HTTP-to-HTTPS
  redirect (1.92.1).
- The Operator supports custom Ingress class names (1.86.0).
- `ProxyClass` accepts recommended annotations while retaining label support
  (1.86.0).
- A `DNSConfig` can assign a static cluster IP to nameservers (1.86.0), and
  nameservers created by a `DNSConfig` default to the stable image (1.92.1).
- The Operator validates ACL tags supplied through `tailscale.com/tags`. It
  also rejects a cluster configuration in which more than one Tailscale
  Kubernetes Service refers to the same Tailscale Service (1.86.0).
- Tailscale Services are generally available, and Operator egress proxies can
  send traffic to their virtual IPs (1.94.1).

## Run and observe the Kubernetes API proxy

Operator v1.86.2 introduces the Kubernetes API proxy and `ProxyGroup` type
`kube-apiserver` for a highly available proxy (1.86.0). Session recording was
beta for `kubectl exec` in v1.82 (1.82.0), and later expanded to `kubectl
attach` and `kubectl debug` (1.86.0).

Operator v1.94.1 adds beta audit logging for events passing through the API
proxy. Audit events can be logged together with, or instead of, complete
session recordings (1.94.1).

Operator v1.96.5 removes `TS_EXPERIMENTAL_KUBE_API_EVENTS`; authorize API
event collection through Tailscale ACLs instead (1.96.2).

For GitOps rendering, `apiServerProxyConfig.mode` and
`apiServerProxyConfig.allowImpersonation` accept either booleans or strings,
which avoids type churn under Argo CD (1.92.1).

## Configure Recorder storage and identity

Recorder Pods can use AWS IRSA rather than static S3 credentials. Configure
the name and annotations of the generated `ServiceAccount` (1.84.0).

A `Recorder` resource can request multiple replicas for high availability,
but a multi-replica deployment must use S3 storage (1.92.1). The default is a
single-replica `StatefulSet` using the filesystem backend (1.96.2).

`tsrecorder` v1.92.3 can load its authentication key from the path named by
`TS_AUTHKEY_FILE` (1.92.1):

```console
export TS_AUTHKEY_FILE=/run/secrets/tailscale-auth-key
```

## Authenticate and isolate Operator deployments

Operator v1.92.3 supports provider-native identity-token authentication.
Operator v1.92.4 fixes Helm rendering when no OAuth client secret is supplied
(1.92.1).

Operator v1.96.5 adds a `Tailnet` custom resource for access to multiple
tailnets and a `ProxyGroupPolicy` custom resource for controlling ProxyGroup
creation by namespace. Ingress and egress ProxyGroup Pods can request a fresh
auth key when needed (1.96.2).

Container image v1.86.2 clears Pod-specific state whenever it starts inside
Kubernetes, preventing identity state from leaking across Pod instances
(1.86.0).
