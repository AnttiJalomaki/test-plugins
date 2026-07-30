# Queries, APIs, and patterns

## Query response formats and parsers

- The query API can return Parquet beginning in 3.4.0, allowing direct
  consumption by columnar tooling.
- LogQL comparisons can use zero-byte values in 3.4.0.
- The query-path `json` parser no longer risks corrupting log lines in 3.5.0.

## Range-query correctness

Corrections in 3.5.0 apply offsets properly to:

- `last_over_time`;
- `first_over_time`;
- `quantile_over_time`.

The query engine also maps `approx_topk` in all cases. In 3.7.3, range-query
evaluation timestamps align to the step grid, and the engine no longer
silently drops `OR` operations.

## Routing and request restrictions

- Label-values queries work with `server.http_path_prefix` in 3.5.0.
- Aggregated metric queries are accepted only from the Logs Drilldown
  application in 3.5.0.
- `query_range` requests can disable caching as of 3.7.3.
- Interval-limit violations return HTTP 400 in 3.7.0.

Clients should handle these status and routing rules explicitly instead of
assuming that every syntactically valid query is accepted on every endpoint.

## Logs Drilldown

Logs Drilldown gains the following in 3.6.0:

- a configuration endpoint;
- partial metric-query results;
- `unwrap` as a projection.

## Persisted patterns

Behind a feature flag in 3.6.0, patterns can be persisted as aggregated
metrics and queried later. Configure volume and frequency bounds. The pattern
ingester supports volume-based filtering and emits detected level as
structured metadata.

The Patterns API accepts multi-tenant queries in 3.7.0.

## Applied-limits API

The tenant applied-limits endpoint added in 3.6.0 returns effective configured
limits. It can filter the response through an allowlist. A request for a
nonexistent tenant returns default limits, so absence of tenant-specific
configuration is not represented by a not-found response.

## Operational UI API routing

The Operational UI's JavaScript moved to a Grafana plugin in 3.6.0, while its
server APIs remain in Loki. Enabling the UI through Helm enables the APIs on
queriers, and the gateway forwards UI requests to them.

## Query attribution

Ruler-originated queries include rule name and rule type in their query tags
as of 3.4.0. Use these tags to distinguish rule evaluation from interactive
and application traffic.

## Scheduler execution

In 3.7.0, the scheduler:

- accounts for total compute capacity;
- shares worker threads across all scheduler connections.

Both are breaking engine changes. Reassess worker counts, fairness, saturation,
and performance baselines rather than carrying forward per-connection thread
assumptions.

## Ruler validation

The ruler validates remote-write configuration in 3.7.0. Treat validation
failures as configuration errors to fix rather than relying on the ruler to
start with an unusable remote-write target.
