# Ingestion, labels, and structured metadata

## Time-sharded ingestion

Per-tenant time sharding is available in 3.4.0:

```yaml
shard_streams:
  time_sharding_enabled: true
```

Loki injects `__time_shard__`. Each resulting stream spans no more than half
of `max_chunk_age`, normally one hour, so long out-of-order ingestion does not
reject very old logs merely for falling too far behind. The Loki Operator can
enable this behavior as of 3.5.0.

## Structured metadata extraction

Ingest-time extraction in 3.4.0 can source structured metadata from:

- ordinary labels;
- existing structured-metadata keys;
- fields parsed from `logfmt` log lines;
- fields parsed from JSON log lines.

In 3.6.0, JSON structured-metadata strings are unescaped. Loki also suppresses
duplicate metadata when the same value comes from a stream label and an
extracted field.

Parsed labels no longer override same-named structured metadata in 3.7.0. This
breaking precedence change should be tested with collision cases.

## Log-level and detected-field behavior

- OTLP ingestion adds `deployment.environment.name` to the default label set
  in 3.4.0.
- The Operator places the log-level attribute into structured metadata in
  3.4.0.
- Automatic log-level discovery detects nested JSON fields and removes colons
  from detected levels in 3.5.0.
- Detected fields recognize byte units in 3.4.0.
- Detected labels accept numeric boolean values in 3.6.0.
- The pattern ingester emits detected level as structured metadata in 3.6.0.

## Distributor enforcement

Starting with 3.5.0, limits can be enforced in distributors or evaluated there
in dry-run mode. Aggregated metric streams are exempt from ordinary label
enforcement. Rate-limit reasons identify stream labels rather than only a
hash, and OTLP entry-metadata bytes count correctly toward limits.

In 3.6.0:

- `MaxRecvMsgSize` controls the distributor's uncompressed message-size
  limit;
- truncated log lines receive an identifier;
- stream limits can be overridden per ingestion policy;
- default policy mappings merge with per-tenant mappings rather than being
  replaced by them.

## Push and service-name semantics

- A push request with no streams returns HTTP 422 in 3.4.0. Clients should not
  expect an empty push to succeed.
- An automatically discovered `service_name` is retained for retention
  decisions and usage tracking in 3.4.0.

## Lambda-promtail

Lambda-promtail supports Prometheus-style relabel configuration in 3.4.0.
Use relabel rules to mutate or filter entries before sending them to Loki. Its
Terraform deployment also exposes an S3 bucket-notification filter-prefix
variable.

Lambda-promtail is not covered by Promtail's deprecation or later removal.

## Fluent integrations

As of 3.6.0, Fluent Bit v4's `out_loki` plugin can send structured metadata.
The Fluentd plugin accepts:

```text
compress gzip
```

## Internal labels

The internal `__aggregated_metric__` label is hidden from `/series` and
`/labels` in 3.6.0. Do not build clients that require that implementation
label to appear in discovery endpoints.
