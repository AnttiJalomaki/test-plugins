# Clustering, configuration, and operations

Use this reference for peer discovery, cluster formation, health signals,
limits, resource alarms, and node-level operations.

## Contents

- [Cluster formation](#cluster-formation)
- [Quorum-aware shutdown](#quorum-aware-shutdown)
- [Metadata-store readiness](#metadata-store-readiness)
- [Client readiness](#client-readiness)
- [Resource alarms](#resource-alarms)
- [Operating-system limits](#operating-system-limits)
- [Queue-member crash logging](#queue-member-crash-logging)
- [Cluster-wide declaration limits](#cluster-wide-declaration-limits)
- [Stream replication networking](#stream-replication-networking)

## Cluster formation

### Discovery retries

`cluster_formation.discovery_retry_limit` accepts positive integers or
`infinity`:

```ini
cluster_formation.discovery_retry_limit = infinity
```

### Consul

Use Consul for discovery without registering RabbitMQ services when another
system such as Nomad owns registration:

```ini
cluster_formation.registration.enabled = false
```

### Kubernetes

The Kubernetes peer-discovery plugin no longer depends on the Kubernetes API.
During first formation, it tries to join the node indexed at `0` as the seed.
This remains backward compatible.

### AWS

Starting in 4.1.7, AWS peer discovery uses IPv6 discovery endpoints in
IPv6-only environments.

### Khepri and Mnesia

- Khepri's default cluster-formation timeout is five minutes, matching
  Mnesia.
- A reset former Mnesia cluster member attempts to leave the old cluster and
  retries joining it, matching the Khepri behavior.

## Quorum-aware shutdown

Before stopping a node, determine whether a quorum queue, stream, or internal
component would lose online quorum:

```shell
rabbitmq-diagnostics check_if_node_is_quorum_critical
rabbitmq-upgrade await_online_quorum_plus_one
```

Use the second command in automation that should wait for quorum plus one.

## Metadata-store readiness

Use separate checks for initialization and initialization with data:

```shell
rabbitmq-diagnostics check_if_metadata_store_is_initialized
rabbitmq-diagnostics check_if_metadata_store_is_initialized_with_data
```

HTTP equivalents:

```text
GET /api/health/checks/metadata-store/initialized
GET /api/health/checks/metadata-store/initialized/with-data
```

## Client readiness

Starting in 4.1.1:

- `GET /api/health/checks/below-node-connection-limit` succeeds while the node
  is below its AMQP/AMQPS connection limit.
- `GET /api/health/checks/ready-to-serve-clients` also requires the node to be
  booted and outside maintenance mode.
- Protocol-listener checks accept comma-separated protocol names.

The original all-in-one HTTP health endpoint is a no-op rather than the former
mega-check. Use the focused endpoints.

## Resource alarms

- Starting in 4.1.6, MQTT, STOMP, and Web MQTT connections remain blocked
  until all memory and disk alarms clear.
- Direct AMQP 0-9-1 Shovel connections within a cluster are blocked by alarms
  like network Shovel connections.
- The latter behavior applies to direct connections, not the AMQP 1.0-based
  `local` Shovel transport.

## Operating-system limits

Starting in 4.1.4, the `rabbitmq-server` startup script on Linux, macOS, and
BSD honors `RABBITMQ_MAX_OPEN_FILES`. It can raise a low soft limit when the
hard limit is already sufficient, but it does not replace operating-system
hard-limit configuration.

## Queue-member crash logging

Use `log.summarize_process_state` and `log.error_logger_format_depth` to limit
queue-member state written after abnormal termination. This prevents
allocation spikes from very large diagnostic output.

## Cluster-wide declaration limits

`cluster_exchange_limit` caps application exchange declarations across the
cluster, including protocol-standard predeclared exchanges. Configure the same
value on every node:

```ini
cluster_exchange_limit = 200
```

Administrators can also disable individual queue types, preventing clients
from declaring new queues or streams of those types.

## Stream replication networking

Use `replica_ip_address_family` in `advanced.config` to select IPv6:

```erlang
[
  {osiris, [{replica_ip_address_family, inet6}]}
].
```

Starting in 4.1.7, this choice can also be made through `rabbitmq.conf`.
