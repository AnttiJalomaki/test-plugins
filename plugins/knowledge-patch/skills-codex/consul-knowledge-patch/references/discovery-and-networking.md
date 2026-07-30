# Discovery and Networking

## Agentless external service monitoring

Consul External Service Monitor can run without a Consul agent on the same
node (1.21.0). It connects directly to Consul servers over one outbound TCP
connection and does not join cluster gossip. This reduces deployment and
networking requirements in constrained environments.

## Multi-port service discovery

Service definitions can use the optional `ports` parameter to register
multiple ports in the catalog (1.22.0). Kubernetes Service sync supports
multi-port Kubernetes Services, and Consul DNS provides a `port` field for
addressing a particular service port.

Treat port names as part of the discovery contract shared by registrations,
Kubernetes Services, DNS clients, and any service-mesh routing that selects a
destination port.

## IPv6 and dual stack

Agents and services on VMs and Kubernetes can use IPv4 or IPv6 addresses
(1.22.0). A single address family per datacenter is recommended. IPv6 is not
supported on OpenShift, Nomad, or ECS for this release.

Envoy bootstrap loopback behavior is address-family aware:

- IPv4-only environments use `127.0.0.1`.
- IPv6 and dual-stack environments use `::1`.
- When the agent bind address is IPv6, `upstream.local_bind_address` defaults
  to `::1`.
- In the same case, `proxy.local_service_address` defaults to `::1`.

Account for these defaults when applications, health checks, or firewall rules
assume an IPv4 loopback address.

## Session-driven health state

Consul sessions can update a health check's state (1.21.0). A session can
therefore directly influence the service-health information exposed through
discovery. When using this behavior, treat the session lifecycle and the
health-check lifecycle as coupled.

## KV key-name validation

The key/value endpoint validates key names in 1.22.0. This is a breaking
security change for clients that previously relied on invalid names.

Before upgrading:

1. Inventory code and automation that write KV keys.
2. Check whether existing naming conventions pass validation.
3. Update invalid writers and migrate affected names.
4. Use `DisableKVKeyValidation` only when validation must be disabled for
   compatibility.

Do not treat the configuration switch as evidence that old key names are valid;
it controls whether the endpoint enforces the validation.
