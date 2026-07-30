# Management API and observability

Use this reference for management serialization, API endpoints, diagnostics,
metric migrations, and administrative tooling.

## Contents

- [HTTP API behavior](#http-api-behavior)
- [Diagnostic commands](#diagnostic-commands)
- [Prometheus additions](#prometheus-additions)
- [Ra metric migration](#ra-metric-migration)
- [Management authorization](#management-authorization)
- [`rabbitmqadmin`](#rabbitmqadmin)

## HTTP API behavior

### Serialization

An empty `channel_details` value is serialized as an object (`{}`), not an
array (`[]`).

### Aggregation pool

`management.delegate_count` controls the worker pool that aggregates HTTP API
response data. It defaults to `5`; nodes with many CPU cores can use a higher
value such as `10` or `16`.

### Static connection information

The HTTP API exposes peer address, TLS details, authentication mechanism, and
other static connection information even when statistics collection is disabled.

### User queues

List queues accessible to a user:

```text
GET /users/{user}/queues
```

### Protected users

The `protected` user tag prevents HTTP API modification or deletion. CLI
operations can still remove the tag or recreate the user.

### API reference authentication

Require authentication for the `/api` reference page:

```ini
management.require_auth_for_api_reference = true
```

### Empty mega-check

The legacy all-in-one HTTP API health check no longer performs its former
comprehensive check. Use focused readiness and metadata-store checks.

## Diagnostic commands

### Message sizes

Estimate the distribution of message sizes flowing through the cluster:

```shell
rabbitmq-diagnostics message_size_stats
```

### Quorum leaders

Check queues matching a virtual host and regular expression:

```shell
rabbitmq-diagnostics check_for_quorum_queues_without_an_elected_leader \
  --vhost "vh-1" "^naming-pattern"
```

`--across-all-vhosts ".*"` checks the whole cluster but can be expensive with
many quorum queues.

### Metadata readiness

```shell
rabbitmq-diagnostics check_if_metadata_store_is_initialized
rabbitmq-diagnostics check_if_metadata_store_is_initialized_with_data
```

## Prometheus additions

- Metrics include labels distinguishing identically named metrics scraped
  from aggregated and per-object endpoints.
- Nodes expose an application-published message-size histogram labeled by
  protocol.
- `queue_identity_info` labels each queue's type and whether the scraped node
  is its leader or follower.
- `/metrics/detailed` can filter queue metrics by queue name.

## Ra metric migration

Update alerts and use a 4.2-compatible quorum-queue Raft Grafana dashboard.

### Aggregated `/metrics` renames

| Old | New |
| --- | --- |
| `rabbitmq_raft_log_snapshot_index` | `rabbitmq_raft_snapshot_index` |
| `rabbitmq_raft_log_last_applied_index` | `rabbitmq_raft_last_applied` |
| `rabbitmq_raft_log_commit_index` | `rabbitmq_raft_commit_index` |
| `rabbitmq_raft_log_last_written_index` | `rabbitmq_raft_last_written_index` |

### Aggregated removals

- `rabbitmq_raft_term_total`
- `rabbitmq_raft_entry_commit_latency_seconds`

### Aggregated additions

- `rabbitmq_raft_num_segments` for internal components
- `rabbitmq_raft_max_num_segments` for the largest quorum-queue segment count
- `rabbitmq_raft_commit_latency_seconds` for internal components
- `rabbitmq_raft_max_commit_latency_seconds` for the highest quorum-queue
  latency

### Per-object and detailed `family=ra_metrics`

- Rename `rabbitmq_raft_term_total` to `rabbitmq_raft_term`.
- Add `rabbitmq_raft_num_segments`.
- Expect more metrics per queue.

## Management authorization

Federation link restarts and Shovel management `DELETE` operations require the
`policymaker` tag.

An HTTP authentication backend can return `deny <Reason>` for disclosure to
AMQP clients when enabled:

```ini
auth_http.authorization_failure_disclosure = true
```

## `rabbitmqadmin`

- The standalone `rabbitmqadmin` v2 is generally available and preferred over
  v1.
- v2 includes commands for automating 3.13.x-to-4.2.x blue-green migrations.
- The management plugin no longer serves the v1 download endpoint. Obtain v1
  from the RabbitMQ `v4.2.x` source branch only for temporary compatibility.
