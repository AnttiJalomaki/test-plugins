# Dashboards, Visualizations, and Reporting

Use this reference for dashboard identity and authoring, panels, variables,
transformations, logs and trace presentation, exports, and Enterprise reports.

## Dashboard identity and APIs

- Enterprise Analytics Views deprecates `:dashboardID` routes in favor of
  `uid/:dashboardUID`, and Analytics Summaries deprecates `dashboard_id` in
  favor of `dashboard_uid` (since 11.5.0).
- Dashboard schema validation and a configurable maximum number of displayed
  series arrive in 12.0.0. Users may explicitly render all series.
- Home-dashboard preferences reference the dashboard UID rather than its
  numeric ID (since 12.1.0).
- Deprecated star APIs based on internal IDs are removed (since 12.2.0).
- Annotations saved through a dashboard UID no longer populate the internal
  numeric dashboard ID (since 12.3.0). Consumers must tolerate its absence.
- Deprecated dashboard routes based on internal IDs are removed and
  `/api/dashboards/home` is deprecated (since 12.4.0).
- The Dashboard DTO drops `isStarred`; the mutation API adds annotation CRUD,
  and a panel screenshot API is available (since 13.1.0).

Library panels no longer require unique names (since 12.3.0). Use a stable
identifier instead of assuming a name resolves to one panel.

## Time, variables, and dashboard controls

- Dashboard models and the time picker can define custom quick time ranges
  (since 11.6.0).
- Time regions accept cron syntax for schedules beyond the basic recurring
  controls (since 11.6.0).
- Server-level time-picker quick ranges are configurable, and schema V2
  dashboards transform automatically when exported in `V1Resource` mode
  (since 12.1.0).
- Library-panel RBAC is generally available and enabled by default in 12.1.0;
  remove `libraryPanelRBAC`. Library elements can no longer serve as library
  variables.
- Repeated panels in new layouts support full-screen and embedded solo-panel
  routes. Variables can render below a drop-down, repeating no longer uses
  clone keys, and the Inspect drawer cannot be opened or linked through a URL
  (since 12.2.0).
- Dashboards add an on/off-style `Switch` variable (since 12.3.0).
- Dashboard controls expose annotations; saved time-comparison windows and
  panel time-range edits are available from view mode (since 12.3.0).
- Variable regular expressions can transform display text, and time-series
  dashboards support per-panel filtering (since 12.4.0).
- V1-to-V2 conversion preserves timezone preference and query-variable sort
  mode; a file-defined V2 dashboard can be the home dashboard (since 13.1.0).

## Schema V2, layout, and as-code authoring

Provisioned dashboards support schema V2 and can be edited through their JSON
model (since 12.4.0). Enterprise reporting supports schema V2 dashboards from
12.3.0, and schema-V2 report forms can edit template variables from 12.4.0.

In 13.0.0, dashboard and folder resource APIs graduate to `v1`. Dashboard
`v2` aligns `TransformationKind` and Dashboard Preferences, and API-server
clients may select a preferred resource version.

Grafana 13.0.0 adds an As Code editor with schema validation. Schema-V2
imports may carry labels. Authors can choose a default layout, add rows and
tabs from the side pane, and define section-level variables. Dashboard restore
is enabled by default.

## Transformations

- Extract fields adds Delimiter and RegExp modes, and transformation filtering
  can target multiple query RefIDs (since 11.5.0).
- Variables work in every transformation; unary **Add field from calculation**
  supports `round()` (since 11.6.0).
- Organize fields gains Auto mode, and Regression is generally available
  (since 12.1.0).
- Transpose gains empty-value options; Trend and TimeSeries gain value labels;
  Trend supports a logarithmic x-axis (since 12.2.0).
- Transformations add smoothing (since 12.4.0).

## Canvas, tables, and general visualizations

- Canvas elements support one-click links and actions; visualization actions
  can require confirmation (since 11.6.0).
- Histogram supports multiple native histograms (since 11.6.0).
- Standard datetime units are restricted to millisecond precision (since
  12.0.0).
- State timelines render `false` and empty strings and support value mappings
  for `NaN` and null (since 12.1.0).
- XY charts accept time values on the x-axis, and Tempo service graphs support
  native histograms (since 12.1.0).
- Canvas can hide tooltips on one-click elements and choose connection
  direction dynamically. Pie charts support ascending, descending, or disabled
  sorting (since 12.2.0).
- Tables add frozen columns, maximum row height for variable-height rows, and
  field-sourced tooltips (since 12.2.0).
- Canvas background images may come from non-icon fields. Time series panels
  accept custom x-axis time units (since 12.3.0).
- Table panels render array-valued `FieldType.other` fields as pills, format
  Pill and JSON cells, and attach links or actions to sparkline cells (since
  12.3.0).
- Geomap accepts a MapLibre style as a base layer; previously beta map layers
  are generally available (since 12.3.0).
- Click-and-drag time-range panning is generally available for time series and
  works in candlestick, heatmap, and timeline panels (since 12.4.0).
- Heatmaps gain a linear y-axis. Geomap XYZ tiles accept variables and min/max
  zoom limits (since 12.4.0).
- The Gauge visualization is generally available in 13.0.0, while the
  frontend `Gauge` component is removed from `@grafana/ui`.
- Pyroscope adds a Call Tree visualization (since 13.0.0).

## Logs and trace presentation

- `hide_logs_download` can hide the logs-download action (since 11.6.0).
- Trace-view span filters can be persisted as panel options (since 11.5.0).
- A field selector integrates with Logs and the Logs table, and downloads
  export only selected fields (since 12.3.0).
- The new Logs visualization is enabled by default in 12.4.0. The Logs panel
  supports transformations with infinite scrolling and unwrapped logs with
  optional displayed-field columns. Explore stores log sort order in the URL.
- Dashboard Logs panels add a field selector, persist displayed fields, and
  can hide Level (since 13.0.0). Plugins can supply a custom log grammar, and
  OpenTelemetry log formatting accepts dot-separated label names.
- Grafana recognizes `emergency` as a log level. Missing level becomes
  `unspecified`, distinct from the unrecognized `unknown` level (since
  13.1.0).

## Dashboard data-source composition

Queries using the Dashboard data source can be nested as subqueries inside a
Mixed data source (since 11.5.0). Validate reference IDs when transformations
or alert expressions depend on the nested result.

## Reporting

Enterprise reporting evolves as follows:

- Allowed email domains are configurable, the API server is included by
  default, and internal IDs are deprecated (since 11.5.0).
- Report email subjects are configurable (since 11.6.0).
- Schema V2 dashboards are reportable (since 12.3.0).
- Retries are productized, stabilized PDF rendering no longer uses
  `newPDFRendering`, and schema-V2 report forms can edit template variables
  (since 12.4.0).
- PDF headers can be toggled, footers are configurable, and a report-readiness
  observer is available (since 13.0.0).
- Backend reporting supports URL-based rendering and can restrict report-email
  recipients to organization members (since 13.1.0).

## Removed dashboard and visualization behavior

- The Prometheus query assistant and related components are removed, Loki
  editors no longer expose Resolution, and logs download can be hidden (since
  11.6.0).
- Tempo removes **Aggregate by** (since 12.0.0).
- Library elements can no longer be library variables (since 12.1.0).
- The Datagrid panel is deprecated (since 12.4.0).
- Drilldown Investigations and CSV drag-and-drop snapshot queries are removed
  (since 12.4.0).
- `dashboardScene`, `publicDashboardsScene`, and `logsPanelControls` are
  removed (since 13.1.0).
