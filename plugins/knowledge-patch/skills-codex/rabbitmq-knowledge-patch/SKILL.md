---
name: rabbitmq-knowledge-patch
description: RabbitMQ
version: 4.3.0
license: MIT
metadata:
  author: Nevaberry
---


# RabbitMQ Knowledge Patch

Use this skill when upgrading, configuring, operating, securing, or integrating
RabbitMQ, especially when the work touches Khepri, quorum queues, streams,
AMQP 1.0, MQTT, federation, Shovels, the management API, or Prometheus.

Prefer the deployed server and client versions, effective configuration,
enabled plugins and feature flags, and observed tests over generic guidance.
Treat mixed-version clusters as a short-lived rolling-upgrade state.

## Reference index

| Reference | Topics |
| --- | --- |
| [references/migrations-and-deprecations.md](references/migrations-and-deprecations.md) | Upgrade paths, feature flags, Khepri transitions, removed settings and facilities |
| [references/queues-streams-and-exchanges.md](references/queues-streams-and-exchanges.md) | Quorum and classic queues, stream filtering, delivery limits, retries, exchange controls |
| [references/protocols-and-clients.md](references/protocols-and-clients.md) | AMQP 1.0 and 0-9-1, MQTT, STOMP, WebSocket, Erlang client behavior |
| [references/security-and-authentication.md](references/security-and-authentication.md) | Authentication backends, OAuth/OIDC, LDAP, TLS, permissions, protected resources |
| [references/clustering-configuration-and-operations.md](references/clustering-configuration-and-operations.md) | Peer discovery, cluster formation, health checks, resource alarms, node operations |
| [references/management-api-and-observability.md](references/management-api-and-observability.md) | HTTP API behavior, diagnostics, Prometheus changes, management UI and tooling |
| [references/federation-shovels-and-extensions.md](references/federation-shovels-and-extensions.md) | Federation, local Shovels, interceptors, event exchange, commercial extensions |

## Breaking changes and upgrade blockers

### Move fully to Khepri before the next major upgrade

- Enable all stable feature flags before an upgrade.
- Move a Mnesia-backed cluster to Khepri before completing the upgrade. If the
  first upgraded node sees Mnesia, it attempts migration during boot.
- Do not attempt an in-place upgrade of a Khepri-enabled 3.13 cluster to 4.x;
  its stored format is incompatible. Use a blue-green migration.
- Remove Mnesia-era partition-handling options. The compatibility keys that
  remain accepted are inert.
- Do not use `rabbitmqctl force_reset` in Khepri workflows.

### Follow the supported hop and rolling rules

- Upgrade to the newest series only from its immediate predecessor. Older
  deployments must pass through the required intermediate series.
- Keep adjacent versions mixed only while performing a rolling upgrade and
  finish within hours. New-series features remain unavailable until all nodes
  are upgraded.
- Do not use grow-then-shrink for a whole-cluster upgrade. It changes replica
  identities and can copy large amounts of data; reserve it for replacing one
  node.
- Before stopping a node, verify quorum safety:

```shell
rabbitmq-diagnostics check_if_node_is_quorum_critical
rabbitmq-upgrade await_online_quorum_plus_one
```

### Remove denied or obsolete queue behavior

- Classic queue v1 storage is gone. An `x-queue-version` of `1`, or any
  `x-queue-mode`, makes a declaration fail. Convert existing queues beforehand.
- Non-durable, non-exclusive classic queues are rejected by default. Prefer
  durable queues, exclusive transient queues, or durable queues with a TTL.
- The `amqp_address_v1`, `amqp_filter_set_bug`, `global_qos`, and
  `queue_master_locator` deprecated features require an explicit permit;
  `ram_node_type` has been removed.
- The old stream-retention CLI command is a no-op. Set stream retention with a
  policy.

### Remove stale configuration

- Delete the removed etcd TLS keys and ineffective `*.cacerts` keys; keep
  `cacertfile` where needed.
- Do not configure `tcp_listen_options.buffer` for AMQP listeners because the
  broker auto-tunes that user-space buffer. Kernel `recbuf` and `sndbuf`
  settings still apply.
- Replace the legacy all-in-one HTTP health check with focused health checks.
- Move automation to the standalone `rabbitmqadmin` v2. The management plugin
  no longer serves the v1 download endpoint.

## High-value queue and stream behavior

### Use strict quorum-queue priorities deliberately

Quorum queues have 32 strict priority levels: every higher-priority message is
delivered before lower-priority messages. Revisit capacity assumptions made
for the earlier two-level, 2:1 interleaving scheme, because a steady
high-priority workload can now delay lower priorities.

### Configure native delayed retries

Set `x-delayed-retry-type` to `all`, `returned`, or `failed`, then set
`x-delayed-retry-min` and `x-delayed-retry-max` in milliseconds. Policy keys
omit the `x-` prefix. Delay is:

```text
min(delayed-retry-min * delivery-count, delayed-retry-max)
```

An AMQP 1.0 modified outcome can override the schedule using the Unix
millisecond `x-opt-delivery-time` annotation.

### Distinguish requeues from failed delivery

Quorum queues increment `acquired-count` on every requeue, but increment
`delivery-count` only for failed attempts. Poison-message limits use
`delivery-count`, so non-failed returns do not necessarily consume the limit.
Treat rejects, failed modified outcomes, `basic.reject`, and client loss as
failures; ordinary releases, non-failed modifications, `basic.nack`, consumer
timeouts, and suspect consumer nodes are non-failures.

### Understand the two consumer timeouts

- Delivery acknowledgement timeout precedence is consumer argument, queue
  argument, policy, then global configuration. It applies to quorum and
  commercial JMS queues, not classic queues or streams.
- A separate disconnected-consumer timeout controls how long a quorum queue
  waits after a consumer node becomes unreachable. Its default is 60 seconds
  and it can be set globally, by policy, or per queue.

### Filter streams at the broker

AMQP 1.0 stream consumers can filter `properties` and
`application-properties`, with at most 16 properties in a filter. SQL filters
can inspect message fields and application properties; combine a Bloom-filter
value with SQL to skip chunks before evaluating individual messages.

## Protocol and client compatibility

### Make AMQP 1.0 durability explicit

If a sender omits the AMQP 1.0 header, the message is non-durable. Applications
that require persistence must send a header with `durable=true`.

AMQP 1.0 clients can renew OAuth JWTs without disconnecting, but the broker
closes the connection if renewal is late or fails. Renewed Stream Protocol
tokens are also rechecked against the active virtual host.

### Validate pre-authentication frame sizes

AMQP 0-9-1 client `frame_max` overrides must be at least 8192 bytes. Prefer the
server default of 131072. Node.js `amqplib` clients should use 0.10.7 or later
or explicitly request a larger value.

### Bound MQTT traffic and choose authorization behavior

- MQTT Maximum Packet Size defaults to 16 MiB and must not exceed
  `max_message_size`, which also defaults to 16 MiB.
- `mqtt.disconnect_on_unauthorized` defaults to `true`; set it to `false` to
  retain the connection and return a protocol error.
- MQTT 5 publishers receive `Quota exceeded` in `PUBACK` when a target queue
  rejects a publish at its maximum length.
- Web MQTT bounds decompressed frames, applies `login_timeout`, and can enforce
  configured origin allowlists.

## Security and configuration

### Separate management authentication when needed

Messaging connections and the HTTP API can use different authentication
backend chains:

```ini
auth_backends.1 = ldap
auth_backends.2 = internal
http_dispatch.auth_backends.1 = http
```

If a configured backend belongs to a known but disabled plugin, startup fails.
This avoids a running node with unusable client authentication.

### Audit OAuth, LDAP, and encrypted secrets

- Explicitly configure provider values that older Azure Entra or Auth0 setups
  may have relied on as defaults.
- Use scope aliases and, where supported, variables such as `{vhost}` and
  `{sub}`; configurable discovery endpoints support broader OIDC layouts.
- LDAP nested-group checks are case-insensitive, and LDAP queries can live in
  `rabbitmq.conf`, including multi-line queries.
- Prefix encrypted `default_password` and `ssl_options.password` values with
  `encrypted:`. A colon alone no longer marks a value as encrypted.

### Protect administrative surfaces

- Require authentication for the `/api` reference page.
- Protect virtual hosts, queues, streams, and users where deletion or HTTP API
  modification must be blocked.
- Federation restart and Shovel deletion actions require `policymaker`.
- Credential refresh immediately clears AMQP 0-9-1 permissions and revalidates
  consumers; passive declarations require `configure` permission.

## Operations and diagnostics

### Use focused readiness signals

Distinguish metadata-store initialization from initialization with data.
Client readiness additionally requires a booted node outside maintenance mode
and below its connection limit. Protocol-listener checks accept a
comma-separated protocol list.

### Keep resource alarms conservative

MQTT, STOMP, and Web MQTT remain blocked until every active memory or disk
alarm clears. Direct in-cluster AMQP 0-9-1 Shovel connections are also subject
to alarms; the newer local Shovel transport is a different mechanism.

### Update Raft dashboards

Ra Prometheus metric names and families changed. Update alerts and the quorum
queue Raft dashboard together, accounting for renamed, removed, and new
segment and commit-latency metrics. Use the observability reference for the
exact mappings.

### Prefer the targeted diagnostic commands

```shell
rabbitmq-diagnostics check_for_quorum_queues_without_an_elected_leader \
  --vhost "vh-1" "^naming-pattern"
rabbitmq-diagnostics message_size_stats
rabbitmq-queues force_checkpoint \
  --vhost-pattern "vhost-pattern" --queue-pattern "queue-pattern"
```

Use cross-vhost quorum-leader checks cautiously on clusters with many quorum
queues.
