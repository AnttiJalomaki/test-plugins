# Observability, Alerting, and Dashboards

Use this reference for metrics and traces, Discover and investigations,
Alerting, Anomaly Detection, forecasting, notifications, and SLO views.

## Metrics, traces, and Discover

### ML Commons observability

Since 3.1.0, ML Commons integrates with the OpenSearch metrics framework and
OpenTelemetry-compatible monitoring. It supports runtime instrumentation on
selected code paths and scheduled collection of state-level metrics.

### Trace correlation across custom data

Since 3.1.0, Observability can use custom index names for OpenTelemetry spans,
logs, and service maps, map non-OpenTelemetry log fields, and search traces
across clusters for trace-to-log correlation.

### Trace Analytics compatibility and map limits

Since 3.2.0, Trace Analytics accepts Data Prepper 2.11 OpenTelemetry output.
Dashboards makes the service map's maximum node and edge counts configurable.

### Unified Discover and trace analysis

OpenSearch 3.3.0 adds an optional redesigned Discover interface for log
analytics, distributed tracing, automatic visualization selection, and
context-aware analysis. Discover Traces supports click-to-filter exploration.
Disabled-by-default AI tools add conversational query and visualization
actions.

### Prometheus and APM in Dashboards

Since 3.5.0, Dashboards can query and visualize Prometheus data beside logs and
traces, with PromQL autocomplete and gauge metrics. Its APM interface adds
configuration, service and service-detail views, an application topology map,
and service-correlation drill-downs.

### Agent traces and packaged observability

Since 3.6.0, Agent Traces records agent, language-model, and tool spans through
OpenTelemetry, with a Python instrumentation SDK plus Dashboards DAG and token
usage views. A one-command Observability Stack bundles the collector, Data
Prepper, OpenSearch, Prometheus, and Dashboards. Performance Analyzer adds a
shard-operations collector.

### Metrics exploration and dashboard templating

Since 3.7.0, Dashboards has an Explore Metrics builder that discovers
Prometheus sources and synchronizes generated PromQL with its raw editor.
Dashboard variables substitute `$name` or `${name}`. Visualization
transformations can limit, sort, filter, aggregate, or compute fields without
rerunning the base query.

### Experimental SLO and unified-alert views

OpenSearch 3.7.0 includes an experimental SLO catalog ordered by remaining
error budget, with burn-rate alerts and multi-window evaluation. A unified
alerts view combines monitors and Prometheus rules and renders the Alertmanager
routing tree as read-only.

### Investigation controls

Since 3.6.0, Dashboards investigations can accept a hypothesis, track total and
per-step durations, and rerun log analysis during reinvestigation.

## Anomaly Detection and forecasting

### Feature-level anomaly controls

Since 2.19.0, Anomaly Detection can trigger independently on a feature's rise
or drop and apply per-feature moving suppression rules. An optional structured
result index flattens entity values and arrays for easier queries and
visualizations.

### Native time-series forecasting

Since 3.1.0, Anomaly Detection can build a self-updating forecast from an index
with a timestamp field by incrementally retraining a Random Cut Forest.
Forecasts feed Alerting, and Security provides forecasting roles and
permissions.

### Longer anomaly-detection intervals

Since 3.2.0, Anomaly Detection supports intervals longer than one hour.

### Alerting and anomaly-detection operations

Since 3.3.0, Alerting monitors can use custom user attributes. Anomaly Detection
supports real-time frequency scheduling and a suggest API, with frequency
optional.

### Anomaly Detection daily insights

Since 3.4.0, Dashboards has a Daily Insights page with index management and data
selection. Detectors have an optional auto-create field, and missing-feature
reporting honors detector frequency.

### Alerting and anomaly correlation

Since 3.5.0, alert-trigger execution can enforce access control on result data
in its context. Anomaly Detection can correlate anomalies by temporal-overlap
similarity.

### Alerting and anomaly-detection administration

Since 3.6.0, `plugins.alerting.monitor.max_triggers` caps triggers per monitor,
and Dashboards provides a configurable lookback window for PPL and SQL
monitors. Anomaly Detection detectors can be provisioned and managed with
Terraform.

## Alerting request and scheduling behavior

### Alerting request and payload changes

In 3.1.0, Alerting publishes a list of findings rather than one at a time.
Document-level monitor create and update requests reject index patterns, and a
dry run with an index pattern is prevented.

### Alerting finding publication reversion

OpenSearch 3.2.0 reverts the list publication behavior introduced in 3.1;
Alerting again publishes one finding at a time.

### External alert scheduling

Since 3.7.0, Alerting provides EventBridge Scheduler CRUD and SQS-backed
external monitor scheduling. Configure its two-role EventBridge model with
`execution_role_arn`.

## Notifications

### Mattermost notifications

Since 3.5.0, Mattermost is available both as a notification-channel
configuration type and as a Dashboards notification destination.
