# Ingestion, Labels, and Limits

## Time-sharded ingestion

Since 3.4.0, long out-of-order ingestion can be enabled per tenant:

```yaml
shard_streams:
  time_sharding_enabled: true
```

Loki injects `__time_shard__` so each resulting stream spans at most half of
`max_chunk_age`, normally one hour. This keeps very old entries from being
rejected as too far behind. The Loki Operator can enable the behavior as of
3.5.0.

## Ingest-time structured metadata extraction

A per-tenant feature introduced in 3.4.0 can extract structured metadata from:

- ordinary labels;
- existing structured-metadata keys; and
- keys parsed from JSON and `logfmt` log lines.

The Operator places the log-level attribute in structured metadata, and OTLP
ingestion includes `deployment.environment.name` in its default label set
(3.4.0).

Structured-metadata behavior was refined in 3.6.0:

- JSON structured-metadata strings are unescaped;
- duplicate metadata originating in both a stream label and an extracted
  field is suppressed; and
- detected labels accept numeric boolean values.

The pattern ingester can emit its detected level as structured metadata.

## Lambda-promtail processing

Lambda-promtail accepts Prometheus-style relabel configurations as of 3.4.0.
Use them to mutate or filter entries before sending them to Loki. Its Terraform
deployment also exposes a variable for an S3 bucket-notification filter prefix.

These features belong to Lambda-promtail, which is not subject to Promtail's
deprecation or removal.

## Empty pushes and service discovery

Since 3.4.0, a push request containing no streams returns HTTP 422. Clients
should treat that as an invalid payload rather than a transient server error.

An automatically discovered `service_name` is retained for both retention
decisions and usage tracking. Make retention and accounting tests use the
effective discovered value.

## Distributor limit enforcement

In 3.5.0, limits can be enforced in distributors or checked there in dry-run
mode. Additional limit semantics are:

- aggregated metric streams are exempt from ordinary label enforcement;
- rate-limit reasons identify the stream labels rather than only a hash; and
- OTLP entry-metadata bytes count toward the applicable limits.

In 3.6.0, the distributor adds `MaxRecvMsgSize` for the uncompressed
message-size limit. It also attaches an identifier to log lines it truncates.
Size tests must use the uncompressed request size.

## Ingestion-policy limits

Stream limits can be overridden for each ingestion policy as of 3.6.0. Loki
merges default ingestion-policy mappings with per-tenant mappings rather than
replacing the defaults wholesale. Set only intended tenant differences and
inspect the merged result.

## Kafka ingestion

Kafka-backed ingestion supports tenant-specific topics as of 3.5.0.
Components can consume Kafka records and maintain multiple Kafka clients as of
3.6.0, while the Helm chart exposes `block_builder` configuration for deploying
that path. Coordinate tenant topic mapping, client configuration, and block
builder rollout.

## Log-level and field discovery

Automatic log-level discovery detects nested JSON fields and strips colons
from detected levels as of 3.5.0. Detected fields also recognize byte units,
and LogQL comparisons may use zero-byte values (3.4.0).

Patterns persisted as aggregated metrics can be bounded by volume and
frequency. The pattern ingester also supports volume-based filtering
(3.6.0).

## Internal aggregated-metric labels

The internal `__aggregated_metric__` label is hidden from `/series` and
`/labels` as of 3.6.0. Do not expect discovery endpoints to return it even
though aggregated metric streams are processed internally.

## Label precedence

Parsed labels no longer replace same-named structured metadata as of 3.7.0.
Choose unique names during parsing when both values must remain queryable.
