# HTTP APIs and Observability

## Query and rule APIs

### Query result limits

`/query` and `/query_range` accept a `limit` parameter (since 3.2.0):

```text
/query?limit=100
/query_range?limit=100
```

### Rule-group pagination and state

The rules API can paginate groups (since 3.1.0). Its `groupNextToken` response
field is present even when empty, so clients must not use field absence as the
end-of-pages signal.

An alerting rule that has not yet run has the explicit state `unknown` (since
3.8.0). API and UI clients must accept it with the established states.

### Parse and OpenAPI contracts

`/parse_ast` responses include duration expressions (since 3.12.0). Prometheus
serves an OpenAPI 3.2 HTTP API description at `/api/v1/openapi.yaml` (since
3.10.0).

### Experimental search

Experimental endpoints can search metric names, label names, and label values
(since 3.13.0). With `search-api`, `--web.search.max-limit` caps each request's
`limit` and defaults to 10000 (`feature-flags`). Requests above the cap receive
HTTP 400. The response default of 100 is clamped to a smaller operator cap;
setting the cap to zero allows unbounded requests and is unsafe for untrusted
clients.

## Status APIs

The `/status` response has `Node` and `ServerTime` fields (since 3.2.0).
Dropped targets returned by `/api/v1/targets` carry their scrape-pool name
(since 3.3.0).

`/v1/status/tsdb/blocks` exposes loaded TSDB block metadata (since 3.6.0).
The TSDB status endpoint returns at most 10,000 statistic sets (since 3.9.0), so
clients must not assume exhaustive, unbounded results.

`/api/v1/features` reports server capabilities (since 3.9.0); use it instead of
inferring feature support from a version string. `/api/v1/status/self_metrics`
returns current server self-metric state as JSON (since 3.12.0).

## Query accounting and diagnostics

From 3.13.0, query statistics expose `samplesRead`, plus `samplesReadPerStep`
when `stats=all` and `promql-per-step-stats` are enabled. These count storage
I/O; `totalQueryableSamples` counts samples loaded into evaluation and may count
one reused sample in multiple windows. The engine-wide storage-read counter is
`prometheus_engine_query_samples_read_total`.

Range subqueries no longer evaluate after the parent's last actual step when the
end is not step-aligned. This corrects `peakSamples`, `query.max-samples`, and
storage-read inflation. `totalQueryableSamples` is also corrected after the
first step for an `@`-modified range beneath an at-modifier-unsafe function.

When tracing is enabled, query-log entries include both `traceID` and `spanID`
(since 3.11.0).

## Server and subsystem metrics

### Rules and Go runtime

`rule_group_last_rule_duration_sum_seconds` and
`go_sync_mutex_wait_total_seconds_total` expose rule-group evaluation totals and
Go mutex wait time (since 3.1.0).

### Notifications

`prometheus_notifications_errors_total` increments by the number of affected
alerts, not by one failed notification batch (since 3.1.0). From 3.10.0,
`prometheus_notifications_dropped_total`,
`prometheus_notifications_queue_capacity`, and
`prometheus_notifications_queue_length` have an `alertmanager` label; aggregate
explicitly when older queries expected one unlabeled series.

Each configured Alertmanager has an independent send loop (since 3.10.0), which
changes scheduling across multiple destinations. Notification latency also has
`prometheus_notifications_latency_histogram_seconds` alongside the summary
(since 3.9.0).

### Storage and service discovery

Use `prometheus_tsdb_wal_replay_unknown_refs_total` and
`prometheus_tsdb_wbl_replay_unknown_refs_total` for unknown series references
during replay (since 3.4.0), `prometheus_tsdb_head_stale_series` for stale Head
series (since 3.6.0), and `prometheus_tsdb_sample_ooo_delta` for the accepted or
rejected out-of-order distance in seconds (since 3.9.0).

Most `prometheus_sd_refresh` metrics have a `config` label containing the job
name (since 3.9.0). `prometheus_sd_last_update_timestamp_seconds` records the
last update sent to consumers (since 3.11.0). Per-job
`prometheus_sd_refresh*` and `prometheus_sd_discovered_targets` series are
deleted when their scrape job is removed (since 3.12.0).

### Native-histogram instrumentation

PromQL, rules, service discovery, and scraping export native histograms beside
existing summaries (since 3.9.0). Account for both forms when discovering or
aggregating self-monitoring series.

### Mixin labels

The Prometheus mixin's `cluster` label can be renamed through `clusterLabel`
(since 3.3.0).

## Debug endpoints and UI

`/debug/pprof/fgprof` provides on-demand wall-time profiles (since 3.10.0).

The target UI can display every relabeling step for a discovered target (since
3.8.0), showing how labels changed and why a target was dropped. From 3.12.0,
the Status menu includes a UI for deleting time series and cleaning tombstones.

The `/-/ready` response includes `X-Prometheus-Stopping` in the `NotReady`
shutdown state (since 3.10.0).
