# Queries, APIs, and Command-Line Tools

## Response formats and attribution

The query API can return Parquet responses as of 3.4.0. Use this response
format when results need to flow directly into columnar-data tooling.

`logcli` can opt in to `ProxyFromEnvironment` and includes common labels in
its output (3.4.0). Queries issued by the ruler carry the rule name and rule
type in query tags, which can be used for attribution and analysis.

## LogQL and query-result corrections

LogQL accepts comparisons against zero-byte values, and detected fields
recognize byte units as of 3.4.0.

The following correctness fixes apply as of 3.5.0:

- offsets are applied correctly to `last_over_time`, `first_over_time`, and
  `quantile_over_time`;
- `approx_topk` is mapped in every case; and
- the query-path `json` parser no longer risks corrupting log lines.

In 3.7.0, parsed labels no longer override same-named structured metadata.
This affects the value exposed to queries and is a breaking change.

As of 3.7.3, range-query evaluation timestamps align to the step grid, and the
query engine no longer silently drops `OR` operations. Regression tests should
include step boundaries and both branches of `OR` expressions.

## Routing and request restrictions

Label-values queries work under a configured `server.http_path_prefix` as of
3.5.0. Build clients from the effective prefix rather than assuming the root
path. Aggregated metric queries are accepted only from the Logs Drilldown
application.

The query frontend can resolve IPv6 addresses as of 3.5.0. IPv6 interfaces in
`common.instance_interface_names` are also valid sources for memberlist's
advertise address.

Interval-limit violations return HTTP 400 as of 3.7.0. Empty push payloads are
a separate ingestion error and return HTTP 422.

## Query caching and Patterns API

Starting in 3.7.3, `query_range` requests can disable caching. Use the request
control for freshness-sensitive diagnostics without changing the global cache
configuration.

The Patterns API accepts multi-tenant queries as of 3.7.0.

Patterns can be persisted as aggregated metrics behind a feature flag since
3.6.0. They can be queried later and bounded by volume and frequency. The
pattern ingester supports volume-based filtering and can emit detected log
level as structured metadata.

## Logs Drilldown

As of 3.6.0, Logs Drilldown has:

- a configuration endpoint;
- partial metric-query results; and
- `unwrap` as a projection.

Remember that aggregated metric queries are restricted to this application.

## Applied limits API

A tenant applied-limits endpoint added in 3.6.0 returns the limits configured
for a tenant. It can filter the response through an allowlist. A request for a
tenant that does not exist returns default limits, so absence must not be
inferred from a successful default-valued response.

## Operational UI APIs

The Operational UI JavaScript moved into a Grafana plugin in 3.6.0, while its
server APIs remain in Loki. Enabling the UI through the Helm chart enables the
APIs on queriers, and the gateway forwards UI requests to those queriers.

## `logcli`, `lokitool`, and server commands

In 3.6.0:

- `logcli` gained deletion commands;
- Loki gained the `loki health` command; and
- the ruler rule checker gained namespace-and-group validation.

In 3.7.0, `logcli` can send custom headers. `lokitool` adds regex namespace
filtering, uses the updated ruler path, and accepts alternative TLS environment
variables. Update scripts that hard-code the old ruler path or TLS variable
names.

## Ruler validation

The ruler validates remote-write configuration as of 3.7.0. Treat validation
errors as configuration defects and correct them before rollout.
