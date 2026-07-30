# Data Sources and Observability

Use this reference for query-editor and backend behavior, connectivity and
headers, data-source removals, cloud integrations, logs, traces, metrics, and
profiles.

## Cross-cutting data-source behavior

- Apps can register data sources at runtime rather than relying only on static
  plugin installation (since 11.5.0).
- A Dashboard data-source query can be a subquery inside Mixed (since 11.5.0).
- Data-source queries include dashboard and panel titles as headers (since
  12.1.0).
- Plugin extensions can register data-source configuration components (since
  12.2.0).
- `PUT /api/datasources/uid/:uid` returns HTTP 400 when the payload UID differs
  from the URL UID (since 12.3.0). Keep them identical.
- Name- and internal-ID-based data-source routes are deprecated in favor of
  UIDs (since 12.4.0).
- `forward_user_agent` preserves the client `User-Agent` when forwarding a
  data-source request (since 13.1.0).

## Prometheus and Mimir

- Grafana supports cloud-partner Prometheus sources and enables
  `prometheusRunQueriesInParallel` by default (since 12.0.0).
- The Prometheus query assistant and its related components are removed in
  11.6.0.
- Prometheus emits a deprecation message for Azure authentication (since
  12.1.0).
- Queries containing `$__range` do not use incremental querying (since
  12.2.0).
- Grafana Advisor adds a Prometheus Type Migration check (since 12.3.0).
- In 13.1.0, the core Prometheus integration removes Azure and SigV4
  authentication, and the `grafana-prometheus` package is removed. Migrate
  credentials and imports before upgrading.
- The alerting settings UI exposes Mimir Alertmanager auto-sync configuration
  (since 13.1.0).

## Loki and logs

- Derived fields can use a regular expression with `label`, and Query Builder
  operations can be disabled and re-enabled (since 11.5.0).
- Loki label lookup changes its default from `/series` to `/labels` with a
  `query` parameter (since 11.5.0). Update proxies, access controls, and
  integrations that only permit or emulate `/series`.
- The Loki editor no longer exposes Resolution (since 11.6.0).
- Enterprise caching drops Memcached `reconnect_interval`; Loki drops
  experimental `lokiQuerySplittingConfig` and predefined operations (since
  12.1.0).
- Logs and Logs-table field selectors drive downloads, so exports contain only
  selected fields (since 12.3.0).
- The new Logs visualization is enabled by default in 12.4.0. It supports
  transformations with infinite scrolling and unwrapped logs with optional
  displayed-field columns. Explore persists log sort order in the URL.
- Dashboard Logs panels expose and persist displayed fields and can hide Level
  (since 13.0.0). A plugin may define a custom log grammar, and OpenTelemetry
  label names can contain dots.
- `emergency` is recognized as a log level; absent levels map to `unspecified`,
  not the unrecognized `unknown` category (since 13.1.0).

## Tempo, tracing, and service graphs

- Tempo supports TraceQL Metrics exemplars and applies data-source TLS settings
  to gRPC requests (since 11.5.0). Trace span filters can be stored as panel
  options.
- Tempo supports TraceQL instant metrics and streaming TraceQL metrics results
  (since 11.6.0).
- Tempo gains ad hoc filters and removes **Aggregate by** (since 12.0.0).
- Traces Drilldown is preinstalled, while the external-app Metrics Drilldown is
  generally available and its legacy implementation is removed (since
  12.0.0).
- Tempo service graphs support native histograms (since 12.1.0).
- Jaeger API calls move to its gRPC endpoint. Enterprise Tempo tag and
  tag-value lookup uses backend `CallResource` (since 12.3.0). Both require
  server-side reachability and authentication.
- Tempo stops forwarding incoming and team headers for streaming requests in
  12.4.0, then resumes forwarding them in 13.0.0. Apply the behavior for the
  exact deployed version.
- Trace data sources support VictoriaMetrics for traces-to-metrics (since
  12.4.0).
- Tempo normalizes dynamic integer and double span attributes to `float64` and
  uses a consistent nested span-subframe shape across span sets (since
  13.1.0).

## Zipkin

Zipkin queries move through the Grafana backend in 11.5.0, requiring
server-side network reachability and authentication. Zipkin is removed from
core data-source plugins in 13.1.0; install or replace it explicitly rather
than assuming it remains bundled.

## Pyroscope and profiles

- Explore Profiles is preinstalled on self-hosted Grafana from 11.5.0.
- Pyroscope processes and displays sampling annotations (since 12.2.0).
- Pyroscope series queries support exemplars (since 12.4.0).
- Pyroscope adds a Call Tree, accepts `profileIdSelector`, and includes the
  full label set on exemplars (since 13.0.0).
- Pyroscope supports its heatmap query API (since 13.1.0).

## Elasticsearch

- Field discovery uses `_field_caps`, not `_mapping` (since 11.5.0). Allow
  `_field_caps` in Elasticsearch permissions and proxies.
- Serverless connections, a configurable default query mode, and a raw DSL
  editor arrive in 12.4.0.
- The core Elasticsearch data source is removed in 13.0.0, while the
  separately supplied query editor adds ES|QL and variable-query support.
  Do not assume the data source is bundled.

## CloudWatch

- Logs Insights supports OpenSearch PPL and SQL, accepts an empty `logstimeout`,
  and exposes AWS Amplify Hosting metrics through the updated AWS SDK (since
  11.5.0).
- An unset query region falls back to the configured default region (since
  12.2.0).
- Log Anomalies becomes a query type; the editor recognizes the Logs `diff`
  command with highlighting and completion (since 12.3.0).
- OpenSearch SQL can select log groups with the selector and `$__logGroups`
  macro. Log-group-prefix and all-log queries are available, batch queries are
  generally available, and **Match exact** defaults to false (since 12.4.0).
- CloudWatch Logs results no longer carry data links, while metric-expression
  data links carry an ID (since 13.1.0).

## Google Cloud and Azure

- Azure Monitor adds a logs query builder. Azure Prometheus exemplars are
  generally available and enabled by default, while Basic Logs queries are
  limited to one resource (since 12.0.0).
- Cloud Monitoring supports service-account impersonation, and Azure Resource
  Graph queries can select their scope (since 12.1.0).
- Cloud Monitoring supports Google Cloud `universe_domain` values (since
  12.4.0).
- Google Cloud Monitoring supports Forward OAuth Identity (since 13.1.0).

## InfluxDB

- Influx SQL supports PDC, and ad hoc filters work in raw queries (since
  12.0.0).
- Tag-autocomplete queries may apply a time-range filter (since 12.1.0).
- Ad hoc filters work with expressions, and self-signed CAs are supported
  (since 12.2.0).

## SQL data sources and expressions

- OpenTSDB 2.4 is supported from 11.6.0.
- SQL Expressions enter public preview in 12.2.0.
- PostgreSQL data-source configuration can omit a password so the server
  process can authenticate through `PGPASSFILE` (since 12.3.0).
- MSSQL supports current-user authentication; MySQL and PostgreSQL gain
  variable query editors (since 12.4.0).
- SQL Expressions support `NOT`, and alert expressions may contain a CTE
  (since 12.4.0).
- SQL-expression schema queries interpolate variables (since 13.1.0).

## Actions and query execution

- Grafana Actions supports Infinity authentication (since 12.2.0).
- Server-side expression pipelines isolate failed nodes so independent nodes
  can return partial results (since 13.0.0).
- Math-expression binary operations have a memory limit, and string-to-number
  conversion preserves null and empty strings (since 13.1.0).

## Metrics emitted by Grafana

- `http_response_size_bytes` is exported, and plugin-request metrics include
  the plugin version (since 12.2.0).
- Grafana's HTTP metrics use native histograms by default, with classic
  histograms configurable (since 12.4.0).
- `grafana_alerting_rule_group_rules` includes `folder_uid`, while the HA
  Alertmanager cluster prefix changes (since 12.4.0).
- Enterprise removes duplicate cache metrics
  `grafana_caching_items` and `grafana_caching_size`, and Grafana no longer
  bundles Prometheus dashboards (since 13.0.0).
- External Alertmanager sender metrics identify the data source by UID (since
  13.1.0).
- The dashboard-version metric is removed (since 13.1.0).
