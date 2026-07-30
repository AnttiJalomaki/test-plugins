---
name: consul-knowledge-patch
description: HashiCorp Consul
version: 2.0.0
license: MIT
metadata:
  author: Nevaberry
---


# HashiCorp Consul Operations and Development

Use this skill when upgrading Consul, configuring agents, registering services,
operating service mesh or gateways, integrating Kubernetes or OpenShift, or
building operational automation around Consul APIs and telemetry.

Start with the running cluster's edition, deployment platform, Consul version,
Envoy version, datacenter topology, and ACL configuration. Enterprise-only
features must not be proposed for Community deployments.

## Reference index

| Reference | Topics |
| --- | --- |
| [Upgrades and lifecycle](references/upgrades-and-lifecycle.md) | Support cadence, rolling and federated upgrades, license transitions, Autopilot replacement |
| [Discovery and networking](references/discovery-and-networking.md) | Multi-port registration, DNS, IPv6, external service monitoring, sessions, KV validation |
| [Service mesh and gateways](references/service-mesh-and-gateways.md) | Envoy compatibility, gateway security, SDS certificates, limits, scaling, passive health |
| [Platforms, identity, and storage](references/platforms-identity-and-storage.md) | Kubernetes, OpenShift, Pod Security Admission, OIDC, cloud snapshot authentication |
| [Operations and telemetry](references/operations-and-telemetry.md) | HTTP timeouts, RPC limits, utilization reporting, product and certificate telemetry, licensing |

## Breaking changes and required migrations

### Treat KV key validation as a compatibility boundary

Consul 1.22.0 validates key names at the key/value endpoint. Applications that
previously wrote invalid names can fail after the upgrade. Audit writers and
stored key conventions before rollout. `DisableKVKeyValidation` can disable the
new validation when temporary compatibility is required.

### Migrate incompatible OpenShift gateway resources

For OpenShift 4.19 and later, use the newer resource types in the
`consul.hashicorp.com` API group. Earlier Kubernetes Gateway API `v1alpha`
resources are incompatible and existing gateway resources must be migrated as
part of the OpenShift upgrade.

### Align Envoy before changing Consul

Consul 1.22.0 bundles Envoy 1.35.3 and no longer supports Envoy 1.31.10.
Consul 2.0.0 expects Envoy 1.37.2 or newer. This is especially important when
Envoy is installed or pinned separately.

For Envoy 1.35 and later, generated configuration includes a TLS transport
socket only when a CA bundle exists. Do not assume the socket is emitted in a
deployment without a CA bundle.

### Preserve gateway intention enforcement

API Gateway and terminating-gateway HTTP listeners normalize request paths.
Keep that behavior enabled so non-normalized paths cannot bypass L7 intention
RBAC checks.

## Upgrade quick reference

### Choose a supported jump

- Routine upgrades should span no more than two major Consul version jumps.
- Community operators generally move to the latest major release about every
  four months.
- Standard Enterprise majors are maintained for about one year.
- Enterprise LTS operators can upgrade about annually, with jumps of at most
  three major versions; LTS releases are supported for about two years.
- Automated replacement still requires old and new versions that support a
  direct upgrade.

### Roll agents in the safe order

1. Restart server agents one at a time.
2. Wait for each server to become healthy and rejoin.
3. Roll client agents only after the server set is healthy.
4. On service-mesh clients, stop Envoy after stopping the old agent and start a
   compatible Envoy after the new agent starts.
5. Confirm builds and protocols with `consul members`.

For WAN federation, complete the primary datacenter before each secondary.
Within a server set, identify the leader with
`consul operator raft list-peers`, upgrade followers first, and the leader last.

### Use two phases for protocol transitions

If release notes require an incompatible protocol transition, first run the new
binary with the previous protocol:

```shell
consul -v
consul agent -protocol=PREVIOUS
```

After every node runs the new binary, restart all agents without the override.
The `-protocol` flag selects the spoken version rather than changing the full
protocol range the agent understands; an older protocol can disable features.

### Account for service and token availability

Between `consul leave` and a client restart, services on that client are
unhealthy and undiscoverable. Provide redundant instances on other clients for
zero-downtime work.

If `enable_token_persistence` was disabled and server tokens are absent from
configuration files, reapply the `agent` and `default` tokens after restart so
the server can rejoin.

## Common configuration changes

### Register and route multiple ports

Service definitions can use optional `ports` data to register multiple catalog
ports. Kubernetes Service sync handles multi-port Services, and Consul DNS can
select a named service port through its `port` field.

Enterprise service-mesh sidecars advertise named local ports with
`proxy.local_service_ports`. An upstream selects one with
`proxy.upstreams[].destination_port`.

- Direct-mode applications continue to dial `localhost:<bind-port>`.
- Transparent-proxy applications can dial
  `<port-name>.<service>.virtual.consul`.

### Configure address families deliberately

Agents and services on VMs and Kubernetes can use IPv4 or IPv6. Prefer one
address family per datacenter. IPv6 is unavailable on OpenShift, Nomad, and ECS
for the relevant release.

Envoy bootstrap uses `127.0.0.1` for IPv4-only environments and `::1` for IPv6
or dual stack. When the agent bind address is IPv6,
`upstream.local_bind_address` and `proxy.local_service_address` default to
`::1`.

### Preserve blocking queries

Agent `http_config.read_timeout` and `write_timeout` default to 15 minutes so
long-polling blocking queries are not cut off early. `read_header_timeout`
remains 10 seconds and `idle_timeout` remains 120 seconds. All four settings
are configurable.

### Harden OIDC without relying on a client secret

PKCE is enabled by default for Consul UI OIDC login. Providers may authenticate
the OIDC client with a JWT assertion instead of a client secret.

## Gateway and mesh quick reference

### Rotate certificates through SDS

API Gateway listeners can use a default SDS TLS certificate. HTTP or TCP route
services may override it and inherit the listener's SDS cluster when the
override omits one. Conflicting override mappings are rejected.

Terminating-gateway upstream TLS also uses SDS, allowing certificate changes
without restarting the gateway.

### Bound API Gateway upstream load

Set gateway-wide upstream defaults and per-route-service overrides for:

- `MaxConnections`
- `MaxPendingRequests`
- `MaxConcurrentRequests`

Enterprise API Gateways on Kubernetes can scale beyond eight replicas and can
enable Horizontal Pod Autoscaling with annotations on the Gateway resource.

### Configure passive failure detection

Enterprise `PassiveHealthCheck` supports `Consecutive5xx`,
`ConsecutiveGatewayFailure`, and `EnforcingConsecutiveGatewayFailure` for Envoy
outlier detection. Gateway failures mean HTTP 502, 503, or 504 responses.

## Operational controls

### Change cluster RPC limits without restarts

Enterprise provides a Raft-replicated `rate-limit` configuration entry for
cluster-wide RPC limits and critical-method exemptions. Discover targetable RPC
method names with `GET /v1/internal/rpc/methods`; the request needs an ACL token
with `operator:read`.

### Monitor certificate expiry

Use `/agent/metrics` Prometheus metrics for active root and signing CAs, agent
TLS certificates, and leaf-renewal health. Dimensions include datacenter,
partition, and namespace. The agent `telemetry` block can emit structured
certificate-monitoring logs with severity thresholds, and the Connect CA API
exposes root and intermediate `NotAfter` values.

### Distinguish collection from export

Census metrics collection is always enabled. License-utilization export remains
configurable, and self-managed Enterprise product-usage telemetry is disabled
by default until explicitly enabled.

## Platform checks

- Agentless Consul ESM connects directly to servers over one outbound TCP
  connection and does not join gossip.
- The Enterprise Kubernetes snapshot sidecar supports local, Amazon S3, Azure
  Blob Storage, and Google Cloud Storage targets.
- Azure Blob snapshots can authenticate with Azure Managed Service Identity.
- Namespace-scoped Kubernetes Pod Security Admission replaces
  PodSecurityPolicy for minimum pod-security enforcement.
- OpenShift compatibility and gateway resource requirements differ by
  OpenShift release; check the platform reference before upgrading.
- Enterprise can use CyberArk Workload Identity Manager, also known as Venafi
  Firefly, as the Connect CA through `connect.ca_provider =
  "pan-distributed-issuer"`.

## Verification checklist

Before a change:

- Record edition, Consul and Envoy versions, datacenters, and platform.
- Check the direct-upgrade path and protocol-transition requirements.
- Audit KV key names, gateway resources, Envoy pins, ACL tokens, and service
  redundancy.
- For Enterprise license transitions, confirm the required server-first order.

After a change:

- Confirm every agent's build and protocol with `consul members`.
- For federation, verify `consul members -wan` and ACL replication from a
  secondary datacenter.
- Confirm gateway certificates, route limits, and normalized-path behavior.
- Check certificate expiry and renewal metrics.
- Confirm utilization and product telemetry export settings match policy.

Use the indexed references for exact rollout sequences, API endpoints,
configuration names, platform limitations, and edition-specific behavior.
