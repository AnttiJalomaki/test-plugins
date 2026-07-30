# Operations, Observability, and Alerting

Use this reference for Query Insights, workload management, metrics and traces,
Dashboards analysis, anomaly detection and forecasting, Alerting and
Notifications, Job Scheduler, ISM, snapshots, replication, and operational
diagnostics.

## Query Insights and live-query operations

### Historical top-N queries

- Query Insights 2.19.0 adds a Dashboards interface for historical top-N
  queries, drill-downs, configuration, and retention.
- The backend adds fetch-by-ID and automatic expiration in 2.19.0.
- The custom local-index-name setting is removed in 2.19.0.
- Query Insights can exclude selected indexes from insight queries and attach
  metric labels to historical data in 3.1.0.
- The Query Insights reader search limit increases to 500 in 3.2.0.
- Top-N query records can include username and user roles in 3.5.0.
- Wrapper endpoints around Query Insights settings provide finer-grained
  access control in 3.5.0.
- The 3.5.0 Top-N dashboard integrates Workload Management groups with
  filtering and sorting.
- Non-admin users in 3.6.0 can be limited to queries authorized by username and
  shared backend roles.
- The 3.6.0 rule-based recommendation service asynchronously analyzes top-N
  queries and returns confidence and estimated impact.
- Top-N data can be exported in 3.6.0 as timestamp-organized JSON to remote
  blob repositories; S3 is supported in that release.
- Dashboards 3.6.0 adds P90/P99 statistics plus distribution, line, and
  heatmap visualizations.
- The 3.7.0 Top Queries API can include recommendations with its
  `recommendations` parameter.

### Live and inflight queries

- Query Insights 3.0.0 adds an inflight/live-queries API for real-time
  monitoring.
- The top-queries API adds `verbose` in 3.0.0, and Dashboards can render its
  returned columns dynamically.
- Live Queries responses include `isCancelled` in 3.1.0.
- Dashboards 3.1.0 adds dedicated Live Queries and Workload Management views.
- Inflight Queries in Dashboards supports multiple data sources in 3.2.0.
- Live Queries can filter by workload group in 3.3.0, with bidirectional
  navigation between group and live-query views.
- Query Insights Dashboards adds version-aware settings in 3.4.0 and
  multiple-data-source support on the Live Queries page.
- Its 3.4.0 Workload Management view can use security attributes.
- Live Queries 3.6.0 adds shard-level task details and an on-demand cache of
  recently finished searches. Failed queries are explicitly tagged.

### Profiling

- Query Insights 3.7.0 adds a Dev Tools profiler with shard-level timing and a
  collapsible query hierarchy, plus navigation from Query Details.

## Workload management

- Index-based auto-tagging in 3.1.0 assigns workload groups through rules, so
  requests need not all carry an explicit header tag.
- Rule-based auto-tagging expands in 3.3.0 from index patterns to principal
  attributes such as username and role.
- Per-group settings in 3.7.0 can override search timeout, cancellation
  interval, maximum bucket count, and other search settings for every request
  routed to the group.

## Metrics, traces, and investigations

### Trace correlation and analysis

- Observability 3.1.0 supports custom index names for OpenTelemetry spans,
  logs, and service maps, mappings for non-OpenTelemetry log fields, and
  cross-cluster trace search for trace-to-log correlation.
- Trace Analytics 3.2.0 accepts Data Prepper 2.11 OpenTelemetry output.
- Dashboards 3.2.0 makes service-map node and edge limits configurable.
- The optional redesigned Discover interface in 3.3.0 unifies log analytics,
  distributed tracing, automatic visualization selection, and context-aware
  analysis.
- Discover Traces 3.3.0 adds click-to-filter exploration. Disabled-by-default
  AI tools add conversational query and visualization actions.
- Dashboards investigations in 3.6.0 accept a hypothesis, track investigation
  and step durations, and can rerun log analysis during reinvestigation.

### Prometheus, APM, and metrics

- Dashboards 3.5.0 can query and visualize Prometheus data with logs and
  traces, with PromQL autocomplete and gauge metrics.
- The 3.5.0 APM interface adds configuration, service and service-detail
  views, an application topology map, and service-correlation drill-downs.
- The 3.6.0 one-command Observability Stack bundles the collector, Data
  Prepper, OpenSearch, Prometheus, and Dashboards.
- Performance Analyzer 3.6.0 adds a shard-operations collector.
- The 3.7.0 Explore Metrics builder discovers Prometheus data sources and
  synchronizes generated PromQL with its raw editor.
- Dashboard variables in 3.7.0 use `$name` or `${name}` text substitution.
- Visualization transformations can limit, sort, filter, aggregate, or
  compute fields in 3.7.0 without rerunning the base query.

### Agent traces

- Agent Traces 3.6.0 records agent, language-model, and tool spans through
  OpenTelemetry, with a Python instrumentation SDK and Dashboards DAG and
  token-usage views.

## Anomaly detection and forecasting

### Detection controls

- Anomaly Detection 2.19.0 can trigger independently on a feature's rise or
  drop and apply moving suppression rules per feature.
- Its optional structured result-index format flattens entity values and
  arrays for querying and visualization.
- Detection intervals longer than one hour are supported in 3.2.0.
- Anomaly Detection 3.3.0 adds real-time frequency scheduling and a suggest
  API; frequency is optional.
- Detectors gain an optional auto-create field in 3.4.0.
- Missing-feature reporting honors detector frequency in 3.4.0.
- Dashboards 3.4.0 adds Daily Insights with index management and data
  selection.
- Anomalies can be correlated by temporal-overlap similarity in 3.5.0.

### Forecasting

- Native forecasting in 3.1.0 builds a self-updating forecast from an index
  with a timestamp field by incrementally retraining a Random Cut Forest.
- Forecasts can feed Alerting, and Security adds forecasting roles and
  permissions.
- Anomaly Detection detectors can be provisioned and managed through Terraform
  in 3.6.0.

## Alerting and Notifications

### Finding and request behavior

- Alerting 3.1.0 publishes a list of findings rather than one finding at a
  time.
- This is reverted in 3.2.0; Alerting again publishes individual findings.
- Document-level monitor create/update requests reject index patterns in
  3.1.0, and dry-run execution with an index pattern is prevented.
- Alerting monitors can use custom user attributes in 3.3.0.
- Alert trigger execution can apply access control to result data exposed in
  its context in 3.5.0.

### Monitor administration

- `plugins.alerting.monitor.max_triggers` caps triggers per monitor in 3.6.0.
- PPL/SQL monitor Dashboards configuration adds a lookback window in 3.6.0.
- PPL monitor CRUD and manually triggered execution are available through
  Alerting in 3.7.0, with RBAC checks on manual runs.
- PPL monitor names can contain up to 100 characters in 3.7.0 rather than 30.

### Destinations and scheduling

- Mattermost is a notification-channel configuration type and Dashboards
  notification destination in 3.5.0.
- Alerting 3.7.0 adds EventBridge Scheduler CRUD and SQS-backed external
  monitor scheduling.
- Configure the two-role EventBridge model with `execution_role_arn`.

### SLO and unified alert views

- The experimental 3.7.0 SLO catalog orders objectives by remaining error
  budget and supports burn-rate alerts and multi-window evaluation.
- The experimental unified alerts view combines monitors and Prometheus rules
  and renders the Alertmanager routing tree read-only.

## Index State Management and rollups

### Transitions and actions

- ISM transitions add `no_alias` and `min_state_age` in 3.2.0.
- ISM index patterns accept exclusion patterns in 3.4.0.
- The 3.5.0 `convert_index_to_remote` action accepts an optional
  `rename_pattern`.
- The 3.5.0 `search_only` action supports reader/writer separation.
- The 3.7.0 ISM Simulate API evaluates every policy transition against live
  index metrics and reports the next state without modifying cluster state.

### Rollups

- Rollups add cardinality metrics and multi-tier rollup support in 3.5.0.

## Analyzer and replication operations

### Analyzer resource reloads

- `_refresh_search_analyzers` adds `reload_cached_resources` in 3.7.0 for
  hot-reloading resources such as Hunspell dictionaries.
- The API works on metadata-write-blocked indexes such as CCR followers in
  3.7.0.

### Cross-cluster replication

- Every CCR REST API accepts `cluster_manager_timeout` in 3.7.0.
- Stop, pause, start, and resume can clear stale persistent tasks in 3.7.0.
- Replication leaves `number_of_replicas` unchanged when a follower uses
  `auto_expand_replicas` in 3.7.0.

## Scheduler and snapshot services

- `IntervalSchedule` accepts seconds as a unit in 3.2.0.
- Job Scheduler 3.2.0 adds REST APIs to list jobs, optionally per node, list
  all locks, or retrieve one lock.
- Job Scheduler adds a Job History Service in 3.3.0.
- Snapshot Management can delete manually created snapshots in 3.3.0.
