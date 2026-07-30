---
name: rabbitmq-knowledge-patch
description: RabbitMQ
version: 4.3.0
license: MIT
metadata:
  author: Nevaberry
---


# RabbitMQ Knowledge Patch

Use this skill when designing, upgrading, configuring, operating, or
troubleshooting RabbitMQ deployments whose behavior may depend on recent
broker, plugin, protocol, CLI, or observability changes.

## How to Use This Skill

1. Identify the deployed RabbitMQ series and patch level.
2. Read the upgrade and removal guidance before changing a cluster.
3. Match the task to the reference index below.
4. Prefer the repository's manifests, configuration, tests, and observed
   runtime behavior when they differ from general guidance.
5. Apply version-qualified notes only when the deployed version includes the
   stated change.
6. Validate commands on a non-production node before automating them.

## Reference Index

| Reference | Topics |
|---|---|
| [upgrades-and-deprecations.md](references/upgrades-and-deprecations.md) | Direct and rolling upgrade paths, Khepri migration, feature flags, removed settings, deprecated commands and queue forms |
| [configuration-auth-and-security.md](references/configuration-auth-and-security.md) | TLS and CA behavior, authentication backends, OAuth/OIDC, LDAP, protected resources, limits, permissions |
| [protocols-clustering-and-federation.md](references/protocols-clustering-and-federation.md) | AMQP, MQTT, STOMP, WebSocket, peer discovery, federation, shovels, message interception |
| [queues-streams-and-messaging.md](references/queues-streams-and-messaging.md) | Queue types, quorum semantics, stream filters, delayed delivery, Single Active Consumer, commercial queue and stream features |
| [operations-observability-and-tooling.md](references/operations-observability-and-tooling.md) | Health checks, diagnostics, HTTP API behavior, metrics, logging, alarms, `rabbitmqadmin` |

## Breaking Changes First

### Plan upgrade hops explicitly

- Upgrade to 4.1 directly from 4.0.x or 3.13.x only after enabling all stable
  feature flags.
- A 3.13 cluster already using Khepri cannot be upgraded in place to 4.x;
  migrate it with a blue-green deployment.
- Upgrade to 4.2 directly from 4.1.x, 4.0.x, or 3.13.x.
- Upgrade to 4.3 only from 4.2.x, with all stable feature flags enabled.
- Khepri is mandatory in 4.3. Enable `khepri_db` before the upgrade when
  possible; otherwise the first 4.3 node migrates metadata during boot.
- Mixed-version clusters are temporary rolling-upgrade states, not steady
  operating modes. Finish them within a few hours.

Read [upgrades-and-deprecations.md](references/upgrades-and-deprecations.md)
before changing cluster membership or metadata-store state.

### Remove or replace obsolete behavior

- Do not rely on Mnesia partition-handling strategies in 4.3; their accepted
  configuration keys have no effect.
- Non-durable, non-exclusive classic queues are rejected by default in 4.3.
  Prefer durable queues, exclusive transient queues, or durable queues with a
  TTL.
- Classic queue v1 declarations fail in 4.3.
- `rabbitmqctl force_reset` is deprecated because it is incompatible with
  Khepri.
- The legacy all-in-one HTTP health endpoint is a no-op. Use focused health
  checks.
- `rabbitmq-streams set_stream_retention_policy` is a no-op. Configure stream
  retention with a policy.
- The management plugin no longer serves the `rabbitmqadmin` v1 download
  endpoint in 4.3. Use v2.

### Recheck client assumptions

- An AMQP 1.0 message with no header is non-durable. Send a header with
  `durable=true` when persistence is required.
- AMQP 0-9-1 clients must use a pre-authentication `frame_max` of at least
  8192 bytes.
- MQTT's default maximum packet size is 16 MiB and must not exceed the broker's
  `max_message_size`.
- `tcp_listen_options.buffer` is ignored because AMQP user-space TCP buffers
  are auto-tuned; kernel `recbuf` and `sndbuf` remain effective.

## High-Value Configuration

### Metadata and cluster formation

```ini
cluster_formation.discovery_retry_limit = infinity
cluster_formation.registration.enabled = false
```

Use infinite discovery retries only when continuous retry is operationally
appropriate. Disable Consul service registration when another system owns it
but retain Consul peer discovery.

Existing Mnesia installations remain on Mnesia after upgrading to 4.2 even
though Khepri is the default for new deployments. Migration is explicit until
Khepri becomes mandatory for 4.3.

### Authentication and management API

Messaging and HTTP API connections can use different authentication chains:

```ini
auth_backends.1 = ldap
auth_backends.2 = internal
http_dispatch.auth_backends.1 = http
management.require_auth_for_api_reference = true
```

Known authentication backends supplied by disabled plugins now cause startup
to fail. Treat this as configuration validation, not a recoverable client
authentication error.

Starting in 4.1.4, only `default_password` and `ssl_options.password` values
prefixed with `encrypted:` are interpreted as encrypted.

### Cluster and exchange limits

```ini
cluster_exchange_limit = 200
exchange_types.local_random.enabled = false
```

`cluster_exchange_limit` includes protocol-standard predeclared exchanges and
must be identical on every node. Disable local-random exchanges when the load
balancer cannot preserve locality.

### Federation and MQTT behavior

```ini
federation.exchanges.connection_close_timeout = 3000
federation.queues.connection_close_timeout = 3000
mqtt.disconnect_on_unauthorized = false
```

Federation close timeouts are milliseconds and cannot exceed 5000. The MQTT
authorization setting defaults to `true`; setting it to `false` keeps the
connection open and returns the protocol-level error.

## High-Value Queue and Stream Features

### Quorum queue delayed retries

Configure `x-delayed-retry-type` as `all`, `returned`, `failed`, or `disabled`,
plus `x-delayed-retry-min` and `x-delayed-retry-max` in milliseconds. Policy
keys omit the `x-` prefix.

The computed delay is:

```text
min(delayed-retry-min * delivery-count, delayed-retry-max)
```

An AMQP 1.0 modified outcome can override the delay per message with the
Unix-millisecond `x-opt-delivery-time` annotation.

### Strict priorities

Quorum queues support 32 strict priority levels in 4.3. Every
higher-priority message is delivered before lower-priority messages; do not
expect the earlier two-level 2:1 interleaving behavior.

### Stream filtering

- AMQP 1.0 property and application-property filters preserve order while
  selecting different stream subsets.
- A filter can inspect at most 16 properties.
- Broker-side SQL filters can inspect standard fields and application
  properties; combine them with a Bloom-filter value to skip chunks first.

### Consumer timeouts

Quorum and Tanzu JMS queues evaluate consumer timeouts. The precedence is:

1. Consumer argument `x-consumer-timeout`
2. Queue argument `x-consumer-timeout`
3. Policy key `consumer-timeout`
4. Global `consumer_timeout`

The global default remains 1,800,000 ms. Classic queues and streams do not
evaluate this timeout.

## Operational Checks

Before stopping a node:

```shell
rabbitmq-diagnostics check_if_node_is_quorum_critical
rabbitmq-upgrade await_online_quorum_plus_one
```

To distinguish metadata-store states:

```shell
rabbitmq-diagnostics check_if_metadata_store_is_initialized
rabbitmq-diagnostics check_if_metadata_store_is_initialized_with_data
```

To find matching quorum queues without a leader:

```shell
rabbitmq-diagnostics check_for_quorum_queues_without_an_elected_leader \
  --vhost "vh-1" "^naming-pattern"
```

Use `--across-all-vhosts ".*"` cautiously on clusters with many quorum queues.

To compact matching quorum queue logs:

```shell
rabbitmq-queues force_checkpoint \
  --vhost-pattern "vhost-pattern" \
  --queue-pattern "queue-pattern"
```

To estimate cluster message-size distribution:

```shell
rabbitmq-diagnostics message_size_stats
```

## Verification Checklist

- Confirm the exact broker, Erlang/OTP, plugin, and client versions.
- Confirm all stable feature flags before each supported upgrade hop.
- Check quorum-plus-one before node shutdown.
- Confirm metadata-store readiness and intended Khepri state.
- Search configuration for removed, ignored, or no-op settings.
- Exercise AMQP durability, frame size, filter, and routing assumptions.
- Exercise MQTT packet limits, authorization errors, and WebSocket origins.
- Update dashboards for renamed or removed Ra metrics.
- Validate authentication plugins and explicit OAuth provider settings.
- Clear management UI browser state after an upgrade if stale assets fail.
