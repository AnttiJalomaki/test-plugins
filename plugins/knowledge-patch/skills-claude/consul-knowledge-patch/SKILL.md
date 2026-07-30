---
name: consul-knowledge-patch
description: HashiCorp Consul
version: 2.0.0
license: MIT
metadata:
  author: Nevaberry
---


# HashiCorp Consul

Use this guide when changing, upgrading, or operating Consul agents,
service discovery, service mesh, gateways, snapshots, or Enterprise
controls.

## Start Here

1. Inspect the deployed Consul version, edition, datacenter topology, and
   orchestrator before proposing configuration.
2. Record separately pinned Envoy versions and determine whether transparent
   proxying, gateways, WAN federation, or ACL replication are in use.
3. Read the reference file that matches the task; combine references for
   upgrades that cross operational domains.
4. Prefer the running configuration, API responses, manifests, and tests when
   they disagree with generic guidance.
5. Treat edition-specific behavior as unavailable unless Enterprise is
   confirmed.
6. Preserve quorum during every server rollout and validate health between
   nodes.

## Reference Index

| Reference | Read when working on |
| --- | --- |
| [references/upgrades-and-rollouts.md](references/upgrades-and-rollouts.md) | Upgrade cadence, server/client order, federation, protocol transitions, licenses, and Autopilot |
| [references/discovery-networking-and-mesh.md](references/discovery-networking-and-mesh.md) | Service registration, DNS, IP families, ESM, Envoy, and sidecar routing |
| [references/gateways-security-and-identity.md](references/gateways-security-and-identity.md) | Key validation, OIDC, path normalization, certificates, gateway limits, and mesh CA providers |
| [references/kubernetes-and-snapshots.md](references/kubernetes-and-snapshots.md) | Kubernetes, OpenShift, Pod Security Admission, gateway resources, scaling, and snapshot storage |
| [references/enterprise-operations-and-telemetry.md](references/enterprise-operations-and-telemetry.md) | Support, rate limits, utilization, licensing, product telemetry, and certificate monitoring |

## Breaking Changes and Migration Hazards

### Validate KV key names

Expect key/value endpoint requests to reject invalid key names. Audit writers,
automation, and migrations before upgrading. Use `DisableKVKeyValidation` only
as an explicit compatibility decision; disabling validation retains the
previously permissive behavior.

### Normalize gateway paths

Expect API Gateway and terminating-gateway HTTP listeners to normalize request
paths before L7 intention RBAC evaluation. Test clients that depend on unusual
or non-normalized paths and keep policy checks aligned with normalized paths.

### Migrate OpenShift gateway resources

For OpenShift 4.19 or later, replace incompatible earlier Kubernetes Gateway
API `v1alpha` resources with the resource types in the
`consul.hashicorp.com` API group. Include this resource migration in the
OpenShift upgrade plan.

### Check Envoy compatibility

Treat Envoy as a separately versioned dependency whenever it is pinned outside
Consul. Consul that bundles Envoy 1.35.3 no longer supports Envoy 1.31.10.
Consul with the newer mesh compatibility requires Envoy 1.37.2 or newer.
Confirm a shared supported Envoy version before rolling agents.

For Envoy 1.35 and later, expect generated configuration to add a TLS transport
socket only when a CA bundle exists. Do not diagnose the missing socket as a
generation failure when no CA bundle is configured.

### Account for longer agent HTTP timeouts

Expect agent `http_config.read_timeout` and `write_timeout` to default to 15
minutes. This permits long-polling blocking queries that the former 30-second
defaults could terminate. Keep `read_header_timeout` at 10 seconds and
`idle_timeout` at 120 seconds unless the deployment explicitly overrides them.
All four settings remain configurable.

### Sequence Enterprise license updates

When adopting the updated `enterprise-standard` license on an eligible
Enterprise upgrade, roll server agents first and then clients. Restart servers
one at a time with the new license, wait for readiness, and only then restart
clients with the new license.

### Use two phases for protocol transitions

When release guidance requires an incompatible protocol transition, first run
the new binary with `-protocol=PREVIOUS`. After every node runs the new binary,
restart all agents without the override. Remember that `-protocol` changes the
spoken version rather than the full understood range and can suppress newer
features.

## Safe Upgrade Quick Reference

### Preserve server quorum

Restart servers one at a time and wait for each to become healthy and rejoin.
In a server set, use `consul operator raft list-peers`, roll followers before
the leader, and confirm `left` and then `alive` around each replacement.

```shell
consul operator raft list-peers
consul members
```

After servers are healthy, roll client agents. A client is unavailable from
`consul leave` until restart, so place redundant service instances on other
clients when zero downtime is required.

### Coordinate service-mesh proxies

After stopping an old client agent, stop its Envoy proxies. Start the new
agent, then restart a compatible Envoy. Enable centralized sidecar and
mesh-gateway configuration before WAN-federated mesh upgrades.

```hcl
enable_central_service_config = true
```

Use `/v1/agent/self` to check reported Envoy compatibility where supported. If
both Consul versions support the installed Envoy, defer the proxy upgrade when
that reduces rollout risk.

### Roll federated datacenters in order

Upgrade primary-datacenter servers and clients before each secondary
datacenter. After all datacenters are complete, verify WAN membership and
query `/v1/acl/replication` on a secondary; the primary reports replication as
disabled even when replication works.

```shell
consul members -wan
curl -s -H "X-Consul-Token: $CONSUL_HTTP_TOKEN" \
  "https://consul-server-0.secondary/v1/acl/replication?pretty"
```

If token persistence was disabled and tokens are absent from server
configuration, restore the `agent` and `default` tokens after restart so the
agent can rejoin.

## High-Value Features

### Register and route multiple ports

Use a service definition's optional `ports` parameter to register multiple
catalog ports. Address a particular port through the Consul DNS `port` field.
Kubernetes Service sync also accepts multi-port Services.

For Enterprise sidecars, advertise named local ports with
`proxy.local_service_ports` and select one from an upstream with
`proxy.upstreams[].destination_port`. Keep direct-mode applications on
`localhost:<bind-port>`; transparent-proxy applications can dial
`<port-name>.<service>.virtual.consul`.

### Configure IP-family behavior deliberately

Agents and services on VMs and Kubernetes can use IPv4, IPv6, or dual-stack
addressing, but prefer one address family per datacenter. Do not plan IPv6 for
OpenShift, Nomad, or ECS where it is unsupported.

Expect Envoy bootstrap to use `127.0.0.1` in IPv4-only environments and `::1`
for IPv6 or dual stack. When the agent bind address is IPv6,
`upstream.local_bind_address` and `proxy.local_service_address` default to
`::1`.

### Use runtime RPC rate limits

In Enterprise, use the Raft-replicated `rate-limit` configuration entry to
change cluster-wide RPC limits without restarting every server. Exempt
critical methods deliberately. Discover targetable names with
`GET /v1/internal/rpc/methods`; the request needs `operator:read`.

### Rotate gateway certificates through SDS

Configure an API Gateway listener with a default SDS TLS certificate and apply
route-service overrides where needed. A route without an override inherits the
listener's SDS cluster; conflicting mappings are rejected. Terminating-gateway
upstream TLS also uses SDS, so certificate changes do not require a gateway
restart.

### Set gateway upstream limits

Apply gateway-wide defaults or route-service overrides for
`MaxConnections`, `MaxPendingRequests`, and `MaxConcurrentRequests`.

### Monitor certificate expiry

Collect certificate-expiration metrics from `/agent/metrics` for root and
signing CAs, agent TLS certificates, and leaf renewal health. Use the
datacenter, partition, and namespace labels when alerting. Configure structured
certificate-monitoring logs and severity thresholds in the agent `telemetry`
block, and inspect root and intermediate `NotAfter` values through the Connect
CA API.

## Operational Guardrails

- Confirm edition gates before recommending Autopilot upgrade migration,
  cluster-wide RPC rate limits, multi-port sidecars, or extended gateway
  scaling.
- Treat census collection and census export as separate controls; collection
  can remain active while reporting export is configurable.
- Keep anonymized product-usage reporting opt-in; it is disabled by default.
- Test session-driven health changes as part of service-health behavior.
- Verify snapshot authentication and target support for the chosen cloud
  rather than assuming all providers use static credentials.
- Read the detailed reference before changing gateway certificate mappings,
  passive health thresholds, or OpenShift resources.
