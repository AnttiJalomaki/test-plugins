# Data sources and observability

Use this reference for query execution, data-source configuration, backend
routing, logs, metrics, traces, profiles, expressions, caching, and Grafana's
own telemetry.

## Cross-cutting data-source behavior

- In 11.5.0, a Dashboard data-source query can be nested as a subquery inside a
  Mixed data source.
- In 11.5.0, core Grafana supports runtime-registered data sources, allowing an
  app to provide a data source without only static installation registration.
- In 12.0.0, label-based access control for data sources is available as a
  self-service public preview.
- In 12.1.0, data-source requests carry dashboard and panel titles as headers.
- In 12.2.0, plugin extensions can register data-source configuration
  components.
- In 12.2.0, Grafana Actions supports Infinity authentication.
- In 12.3.0, `PUT /api/datasources/uid/:uid` returns HTTP 400 when the payload
  UID and URL UID differ. Keep them identical.
- In 12.3.0, Enterprise query caching is disabled when
  `oauthPassThru=true`, preventing per-user OAuth queries from sharing cached
  results.
- In 12.4.0, name- and internal-ID-based routes are deprecated in favor of
  UID routes.
- In 13.1.0, `forward_user_agent` preserves the client `User-Agent`.
  Asynchronous APIs and hooks supersede `datasourceSrv` for frontend access.

## Prometheus and Mimir

- In 11.6.0, the Prometheus query assistant and its related components are
  removed.
- In 12.0.0, Grafana supports cloud-partner Prometheus data sources and enables
  `prometheusRunQueriesInParallel` by default.
- In 12.1.0, Prometheus emits a deprecation message for Azure authentication.
- In 12.2.0, a Prometheus query containing `$__range` does not use incremental
  querying.
- In 12.3.0, Grafana Advisor adds a Prometheus Type Migration check for
  identifying data sources that need migration.
- In 13.1.0, core Prometheus removes Azure and SigV4 authentication, and the
  `grafana-prometheus` package is removed.
- In 13.1.0, external Alertmanager sender metrics identify data sources by UID,
  and the settings UI can configure Mimir Alertmanager auto-sync.

## Loki and logs

- In 11.5.0, Loki derived fields can use a regular expression with `label`.
  Query Builder operations can be disabled and re-enabled.
- In the 11.5.0 breaking change, label lookup defaults to `/labels` with a
  `query` parameter instead of `/series`. Update proxies, permissions, and
  integrations that assume `/series`.
- In 11.6.0, `hide_logs_download` can hide the download control. Loki query
  editors no longer show `Resolution`.
- In 12.1.0, Loki removes experimental `lokiQuerySplittingConfig` and
  experimental predefined operations.
- In 12.1.0, Enterprise caching removes the Memcached
  `reconnect_interval` setting.
- In 12.3.0, a field selector is integrated into Logs and the Logs table.
  Downloads include only selected fields.
- In 12.4.0, the new Logs visualization is enabled by default. The Logs panel
  supports transformations with infinite scrolling and unwrapped logs with
  optional displayed-field columns. Explore stores log sort order in its URL.
- In 13.0.0, Logs dashboard panels can expose a field selector, persist
  displayed fields, and hide Level. Plugins can supply a custom log grammar;
  OpenTelemetry log formatting accepts dot-separated label names.
- In 13.1.0, `logsPanelControls` is removed. `emergency` is recognized as a log
  level; missing level is `unspecified`, distinct from unrecognized `unknown`.

## Tempo, Jaeger, Zipkin, and traces

- In 11.5.0, Tempo supports TraceQL Metrics exemplars and uses data-source TLS
  settings for gRPC. Trace-view span filters can be stored as panel options.
- In 11.5.0, Zipkin requests run through the Grafana backend. Server-side
  network reachability and authentication are therefore required.
- In 11.6.0, Tempo supports instant TraceQL metrics queries and streamed
  TraceQL metrics results.
- In 12.0.0, Tempo supports ad hoc filters and removes **Aggregate by**.
  Traces Drilldown is preinstalled; the external-app Metrics Drilldown
  implementation is generally available and its legacy paths are removed.
- In 12.1.0, Tempo service graphs support native histograms.
- In 12.3.0, Jaeger API calls move to its gRPC endpoint. Enterprise Tempo tag
  and tag-value lookup moves through data-source backend `CallResource`;
  server-side connectivity and credentials are required.
- In 12.4.0, Tempo stops forwarding incoming and team headers on streaming
  calls. VictoriaMetrics can be used for traces-to-metrics.
- In 13.0.0, Tempo again forwards incoming and team headers for streaming,
  reversing the 12.4.0 behavior.
- In 13.1.0, Zipkin is removed from core data-source plugins. Install and
  manage any replacement explicitly.
- In 13.1.0, Tempo converts dynamic integer and double span attributes to
  `float64` and uses one nested span-subframe schema across span sets.

## Pyroscope and profiles

- In 11.5.0, Explore Profiles is preinstalled on self-hosted instances.
- In 12.2.0, Pyroscope processes and displays sampling annotations.
- In 12.4.0, Pyroscope series queries support exemplars.
- In 13.0.0, Pyroscope adds Call Tree, accepts `profileIdSelector`, and attaches
  the complete label set to exemplars.
- In 13.1.0, Pyroscope supports its heatmap query API.

## Elasticsearch

- In 11.5.0, field discovery uses `_field_caps`, not `_mapping`. Permit that
  endpoint in proxies and Elasticsearch roles.
- In 12.4.0, Elasticsearch supports serverless connections, a configurable
  default query mode, and a raw DSL editor.
- In 13.0.0, the core Elasticsearch data source is removed, so it is no longer
  implicitly bundled. The separately installed integration's query editor adds
  ES|QL and variable-query support.

## CloudWatch

- In 11.5.0, the CloudWatch source supports OpenSearch PPL and SQL in Logs
  Insights. Empty `logstimeout` is accepted, and the updated AWS SDK exposes
  AWS Amplify Hosting metrics.
- In 12.2.0, a query with no region uses the configured default region.
- In 12.3.0, CloudWatch Logs adds the Log Anomalies query type and editor
  support for the Logs `diff` command, including highlighting and autocomplete.
- In 12.4.0, OpenSearch SQL can select log groups with the selector and
  `$__logGroups`. CloudWatch adds log-group-prefix and all-log queries; batch
  queries are generally available; **Match exact** defaults to false.
- In 13.1.0, CloudWatch Logs results no longer contain data links. Metric
  expression data links now carry an ID.

## Google Cloud and Azure

- In 12.0.0, Azure Monitor adds a logs query builder; Azure Prometheus
  exemplars are generally available and enabled by default. Basic Logs queries
  support only one resource.
- In 12.1.0, Cloud Monitoring supports service-account impersonation. Azure
  Resource Graph queries allow scope selection.
- In 12.4.0, Cloud Monitoring accepts Google Cloud `universe_domain`.
- In 13.0.0, Azure integrations support certificate authentication.
- In 13.1.0, Google Cloud Monitoring supports Forward OAuth Identity.

## InfluxDB, OpenTSDB, and SQL data sources

- In 11.6.0, OpenTSDB supports OpenTSDB 2.4.
- In 12.0.0, Influx SQL supports PDC, and ad hoc filters work with raw InfluxDB
  queries.
- In 12.1.0, InfluxDB tag autocomplete can optionally apply a time-range
  filter.
- In 12.2.0, InfluxDB ad hoc filters work with expressions, and a self-signed
  CA can be configured.
- In 12.3.0, PostgreSQL data-source configuration no longer requires a
  password, allowing server-side credentials through `PGPASSFILE`.
- In 12.4.0, MSSQL supports current-user authentication. MySQL and PostgreSQL
  add variable query editors.

## SQL and server-side expressions

- In 12.2.0, SQL Expressions enter public preview.
- In 12.4.0, SQL Expressions support `NOT`, and alert expressions may contain a
  CTE.
- In 13.0.0, one broken node no longer prevents unaffected server-side
  expression nodes from returning partial results.
- In 13.1.0, binary math expressions have a memory limit. SQL-expression schema
  queries interpolate variables, and string-to-number conversion preserves
  nulls and empty strings.

## Unified storage database connectivity

- In 11.5.0, Unified Storage supports PostgreSQL `verify-full` and prefers TLS
  when Grafana's database connection uses SSL.
- In 12.1.0, storage honors `migration_locking`, and Unified Storage respects
  `GF_DATABASE_URL`.

## Grafana metrics and runtime observability

- In 12.2.0, Grafana exports `http_response_size_bytes`, and plugin request
  metrics include the plugin version.
- In 12.4.0, Grafana's HTTP telemetry uses native histograms by default; classic
  histograms remain configurable.
- In 13.0.0, Enterprise query caching removes duplicate
  `grafana_caching_items` and `grafana_caching_size` metrics. Grafana also stops
  bundling Prometheus dashboards. Update metric consumers and monitoring
  provisioning.
- In 13.1.0, the dashboard-version metric is removed.

## Query and visualization data behavior

- In 11.6.0, Histogram handles multiple native histograms.
- In 12.1.0, State timeline renders `false` and empty strings and permits value
  mappings for `NaN` and `null`. Regression is generally available; Tempo
  service graphs accept native histograms; XY charts accept time x-values.
- In 12.2.0, tables support field-sourced tooltips and frozen/variable-height
  rows; Trend can use logarithmic x-axis and value labels.
- In 12.4.0, time-range panning extends to candlestick, heatmap, and timeline;
  heatmaps add a linear y-axis and transformations add smoothing.
