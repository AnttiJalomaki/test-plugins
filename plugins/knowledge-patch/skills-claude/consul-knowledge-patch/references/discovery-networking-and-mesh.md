# Discovery, Networking, and Service Mesh

## Register Multiple Service Ports

Service definitions can set the optional `ports` parameter to register
multiple ports in the catalog (since 1.22.0). Kubernetes Service sync accepts
multi-port Kubernetes Services, and Consul DNS provides a `port` field for
addressing a specific registered port.

## Route Named Ports Through the Mesh

Enterprise sidecars can advertise named local ports with
`proxy.local_service_ports` (since 2.0.0). Select a port from an upstream with
`proxy.upstreams[].destination_port`.

The application addressing mode remains significant:

- Direct-mode applications connect to `localhost:<bind-port>`.
- Transparent-proxy applications can connect to
  `<port-name>.<service>.virtual.consul`.

## Plan IPv4, IPv6, and Dual Stack

Agents and services on VMs and Kubernetes can use IPv4 or IPv6 addresses
(since 1.22.0). Prefer a single address family within each datacenter even
where dual stack is possible.

In this behavior set, IPv6 is not supported on OpenShift, Nomad, or ECS.
Account for loopback defaults in generated Envoy configuration:

- IPv4-only environments use `127.0.0.1`.
- IPv6 and dual-stack environments use `::1`.
- When the Consul agent bind address is IPv6,
  `upstream.local_bind_address` and `proxy.local_service_address` default to
  `::1`.

## Deploy Consul ESM Without a Local Agent

Consul External Service Monitor can run without a colocated Consul agent
(since 1.21.0). It connects directly to Consul servers through one outbound
TCP connection and does not join cluster gossip. Use this topology when a
constrained network can reach servers but should not admit another gossip
member.

## Drive Health Checks from Sessions

Consul sessions can update a health check's state (since 1.21.0). Account for
session lifecycle when reasoning about service-health transitions because a
session can now directly affect the reported health.

## Match Envoy to Consul

The bundled Envoy version is 1.35.3 in the behavior introduced with Consul
1.22.0, and Envoy 1.31.10 is no longer supported. For Envoy 1.35 and later,
generated configuration includes a TLS transport socket only when a CA bundle
is present. This prevents startup failure when no CA bundle is configured.

Consul 2.0.0 advances service-mesh compatibility to Envoy 1.37.2 and newer.
Check separately installed or pinned Envoy packages before upgrading Consul.
During a rolling upgrade, use a version supported by both the old and new
Consul agents where possible.

## Tune Passive Upstream Health

Enterprise `PassiveHealthCheck` adds three Envoy outlier-detection settings
(since 2.0.0):

- `Consecutive5xx` counts general HTTP 5xx responses.
- `ConsecutiveGatewayFailure` counts gateway failures: 502, 503, and 504.
- `EnforcingConsecutiveGatewayFailure` controls enforcement for the gateway
  failure threshold.

Distinguish the general 5xx threshold from the gateway-specific threshold when
configuring ejection behavior.
