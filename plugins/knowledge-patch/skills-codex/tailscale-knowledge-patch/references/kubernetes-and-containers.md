# Kubernetes and Containers

## Container startup and authentication

- Container image 1.80.0 can load `TS_SERVE_CONFIG` when HTTPS is disabled for
  the tailnet, provided the configuration defines no HTTPS endpoint.
- Container image 1.86.2 clears pod-specific state whenever it starts in
  Kubernetes. It also improves direct connectivity to `ProxyGroup` pods by
  using external node IP addresses as static endpoints.
- `tsrecorder` 1.92.3 reads an auth key from the file named by
  `TS_AUTHKEY_FILE`:

  ```console
  export TS_AUTHKEY_FILE=/run/secrets/tailscale-auth-key
  ```

- Container image 1.92.3 restores `iptables` operation on hosts without
  `nftables`.
- Container image 1.94.1 supports OAuth and workload identity federation.

## Operator proxy lifecycle and high availability

- Operator-created proxy `ServiceMonitor` objects can carry user labels (since
  1.80.0).
- Operator proxies dynamically reload changed `tailscaled` configuration.
  Changes such as hostnames can take up to a minute to propagate.
- Restarting egress `ProxyGroup` replicas no longer interrupts cluster
  workloads that access tailnet targets (since 1.80.0).
- Operator-managed Ingresses and Tailscale Kubernetes Services can use a
  `ProxyGroup` with multiple active replicas, multiplex applications over the
  replicas, and expose applications across clusters (since 1.84.0).
  Cluster-wide `EndpointSlice` watching enables failover when a cluster has no
  healthy backend.
- In 1.86.0-era behavior, `ProxyClass` supports recommended annotations while
  still accepting labels. The Operator also supports custom Ingress class
  names and a static cluster IP for `DNSConfig` nameservers.
- The Operator validates ACL tags supplied through `tailscale.com/tags` and
  ensures that only one Tailscale Kubernetes Service in a cluster refers to a
  particular Tailscale Service.

## Ingress and DNSConfig

- Operator 1.82.0 can use Let's Encrypt staging certificates for Ingress TLS,
  configured through the ProxyClass APIs. Use staging during initial setup to
  avoid production rate limits.
- An unset path on an Operator-managed Ingress defaults to `/` (since 1.84.0).
- Add `tailscale.com/http-redirect` to an Ingress to enable HTTP-to-HTTPS
  redirects (since 1.92.1).
- The Operator defaults to the stable nameserver image for `DNSConfig`
  resources (since 1.92.1).

## Kubernetes API proxy, recording, and audit

- Kubernetes API proxy session recording initially covers the contents of
  `kubectl exec` sessions and is beta in Operator 1.82.0.
- Operator 1.86.2 introduces the Tailscale Kubernetes proxy. Use a
  `ProxyGroup` of type `kube-apiserver` to run the API server proxy in
  high-availability mode.
- Recording in the 1.86.0 line also covers `kubectl attach` and `kubectl
  debug`.
- Operator 1.94.1 adds beta audit logging for events passing through its
  Kubernetes API server proxy. Audit events can be logged in addition to or
  instead of complete session recordings.
- Operator 1.96.5 removes `TS_EXPERIMENTAL_KUBE_API_EVENTS`. Configure
  Kubernetes API event behavior through Tailscale ACLs.

## Recorder deployment

- Recorder pods can use AWS IRSA instead of static S3 credentials by setting
  the generated `ServiceAccount` name and annotations (since 1.84.0).
- Recorder resources support multiple replicas for high availability (since
  1.92.1). Multiple replicas require S3 storage.
- As of 1.96.2, the `Recorder` CRD defaults to a single-replica `StatefulSet`
  with the filesystem backend.

## Workload identity and chart rendering

- Operator 1.92.3 can authenticate with provider-native identity tokens.
- Operator 1.92.4 fixes Helm rendering when no OAuth client secret is set.
- The Operator accepts either boolean or string values for
  `apiServerProxyConfig.mode` and
  `apiServerProxyConfig.allowImpersonation`, improving Argo CD compatibility
  (since 1.92.1).

## Multiple tailnets and namespace policy

- Operator 1.96.5 adds a `Tailnet` custom resource for access to multiple
  tailnets and a `ProxyGroupPolicy` custom resource for controlling
  ProxyGroup creation by namespace.
- Ingress and egress ProxyGroup pods can request a new auth key when required
  in the 1.96.2 stable line.
