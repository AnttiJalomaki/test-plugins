# Service Mesh and Gateways

## Envoy compatibility

Consul 1.22.0 bundles Envoy 1.35.3 and drops support for Envoy 1.31.10. With
Envoy 1.35 and later, generated configuration contains a TLS transport socket
only when a CA bundle is present. This prevents startup failures when no CA
bundle exists.

Consul 2.0.0 updates service-mesh compatibility to Envoy 1.37.2 and newer.
Check this explicitly when Envoy is installed or pinned independently from
Consul.

## HTTP path normalization

API Gateway and terminating-gateway HTTP listeners normalize request paths in
2.0.0. The normalization closes a route by which non-normalized paths could
bypass L7 intention RBAC checks. Security verification should exercise
equivalent normalized and non-normalized request forms.

## Multi-port service-mesh routing

Enterprise sidecars can advertise named local ports through
`proxy.local_service_ports` (2.0.0). Upstreams select a named destination with
`proxy.upstreams[].destination_port`.

Connection forms differ by proxy mode:

- Direct-mode applications still connect to `localhost:<bind-port>`.
- Transparent-proxy applications can connect to
  `<port-name>.<service>.virtual.consul`.

## SDS-backed certificates

An API Gateway listener can use a default SDS TLS certificate (2.0.0). HTTP or
TCP route services can override that certificate. If a route override does not
supply an SDS cluster, it inherits the listener's SDS cluster. Consul rejects
conflicting override mappings.

Terminating-gateway upstream TLS also uses SDS. Certificates can therefore
change without a gateway restart.

## API Gateway upstream limits

API Gateway supports gateway-wide upstream defaults and per-route-service
overrides for these controls (2.0.0):

- `MaxConnections`
- `MaxPendingRequests`
- `MaxConcurrentRequests`

Use gateway defaults for the common policy and route-service overrides for
workloads that need distinct bounds.

## Kubernetes API Gateway scaling

Enterprise API Gateways on Kubernetes can scale past the former eight-replica
limit (2.0.0). Horizontal Pod Autoscaling can be enabled through annotations on
the Gateway resource.

## OpenShift gateway migration

OpenShift 4.19 and later use new Kubernetes resource types in the
`consul.hashicorp.com` API group (2.0.0). The earlier Kubernetes Gateway API
`v1alpha` resources are incompatible on those OpenShift releases. Migrate
existing gateway resources during the OpenShift upgrade.

## CyberArk WIM certificate authority

Enterprise service mesh can delegate certificate signing to CyberArk Workload
Identity Manager, also called Venafi Firefly (2.0.0). Configure the Connect CA
provider as:

```hcl
connect {
  ca_provider = "pan-distributed-issuer"
}
```

## Passive upstream health thresholds

Enterprise `PassiveHealthCheck` adds three Envoy outlier-detection settings in
2.0.0:

- `Consecutive5xx` counts general HTTP 5xx responses.
- `ConsecutiveGatewayFailure` counts gateway failures: 502, 503, and 504.
- `EnforcingConsecutiveGatewayFailure` controls enforcement of the gateway
  failure threshold.
