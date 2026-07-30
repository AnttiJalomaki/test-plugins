# Operations, Observability, and Tooling

Use this reference for health probes, diagnostics, HTTP API response details,
Prometheus migrations, logging controls, resource alarms, and administrative
tools.

## Health and Readiness

Batch `4.0.6` adds distinct metadata-store initialization checks:

```shell
rabbitmq-diagnostics check_if_metadata_store_is_initialized
rabbitmq-diagnostics check_if_metadata_store_is_initialized_with_data
```

The HTTP equivalents are:

```text
GET /api/health/checks/metadata-store/initialized
GET /api/health/checks/metadata-store/initialized/with-data
```

The original HTTP API “One True Health Check” is now a no-op and does not run
its old all-in-one check. Replace it with focused health checks.

Starting in 4.1.1:

- `GET /api/health/checks/below-node-connection-limit` succeeds while the
  node is below its AMQP/AMQPS connection limit.
- `GET /api/health/checks/ready-to-serve-clients` additionally requires a
  booted node outside maintenance mode.
- Protocol-listener health checks accept comma-separated protocol names.

## Quorum and Message Diagnostics

Check matching quorum queues for an elected leader:

```shell
rabbitmq-diagnostics check_for_quorum_queues_without_an_elected_leader \
  --vhost "vh-1" "^naming-pattern"
```

Use `--across-all-vhosts ".*"` to check the cluster. It can be expensive on a
cluster with many quorum queues.

Estimate the distribution of message sizes passing through the cluster:

```shell
rabbitmq-diagnostics message_size_stats
```

Node-shutdown quorum checks and forced queue checkpoints are documented in
[upgrades-and-deprecations.md](upgrades-and-deprecations.md) and
[queues-streams-and-messaging.md](queues-streams-and-messaging.md).

## HTTP API Behavior and Capacity

An empty `channel_details` value is serialized as an object (`{}`), not an
array (`[]`).

`management.delegate_count` controls the process pool that aggregates data
for HTTP API responses. It defaults to `5`; nodes with many CPU cores can use
a larger value such as `10` or `16`.

RabbitMQ 4.3 adds:

- `GET /users/{user}/queues`
- Static connection information—including peer address, TLS details, and
  authentication mechanism—even when statistics collection is disabled

## Prometheus Metrics

Prometheus metrics can label whether same-named data came from the aggregated
endpoint or a per-object endpoint.

Starting in batch `4.1.0`, nodes expose:

- A histogram of application-published message sizes labeled by protocol
- `queue_identity_info`, labeling queue type and whether the scraped node is
  the leader or a follower

### Ra metric migration

Batch `4.2.0` changes `rabbitmq_raft*` and `rabbitmq_detailed_raft*` metrics.
Update dashboards and alerts, including the RabbitMQ quorum-queue Raft Grafana
dashboard, to a compatible version.

Aggregated `/metrics` renames:

| Previous | Current |
|---|---|
| `rabbitmq_raft_log_snapshot_index` | `rabbitmq_raft_snapshot_index` |
| `rabbitmq_raft_log_last_applied_index` | `rabbitmq_raft_last_applied` |
| `rabbitmq_raft_log_commit_index` | `rabbitmq_raft_commit_index` |
| `rabbitmq_raft_log_last_written_index` | `rabbitmq_raft_last_written_index` |

Aggregated `/metrics` removes:

- `rabbitmq_raft_term_total`
- `rabbitmq_raft_entry_commit_latency_seconds`

Aggregated `/metrics` adds:

- `rabbitmq_raft_num_segments` for internal components
- `rabbitmq_raft_max_num_segments` for the largest quorum-queue segment count
- `rabbitmq_raft_commit_latency_seconds` for internal components
- `rabbitmq_raft_max_commit_latency_seconds` for the highest quorum-queue
  latency

Per-object and detailed `family=ra_metrics` output:

- Renames `rabbitmq_raft_term_total` to `rabbitmq_raft_term`
- Adds `rabbitmq_raft_num_segments`
- Exposes more metrics for each queue

In 4.3, `/metrics/detailed` can filter queue metrics by queue name.

## Logging

RabbitMQ 4.2 adds `log.summarize_process_state` and
`log.error_logger_format_depth` to limit queue-member state logged after an
abnormal termination. Use them to avoid allocation spikes from extremely
large crash diagnostics.

## Resource Alarms

Starting in 4.1.6, MQTT, STOMP, and Web MQTT connections remain blocked until
all active memory and disk alarms clear. Clearing only one of multiple alarms
does not unblock them.

Direct in-cluster AMQP 0-9-1 shovel connections are also blocked by resource
alarms in 4.2, like network shovel connections. The separate `local` shovel
protocol does not use this behavior.

## Administrative Tooling

`rabbitmqadmin` 2.0 is generally available as a standalone binary and is
recommended over the original tool.

Starting in 4.1.2, force a chosen stream Single Active Consumer member active:

```shell
rabbitmq-streams activate_stream_consumer \
  --stream "stream-name" \
  --reference "consumer-reference"
```

The management plugin's v1 download endpoint is removed in 4.3; see
[upgrades-and-deprecations.md](upgrades-and-deprecations.md).
