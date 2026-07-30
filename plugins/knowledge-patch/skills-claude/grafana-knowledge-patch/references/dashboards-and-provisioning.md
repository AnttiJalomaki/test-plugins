# Dashboards and provisioning

Use this reference for dashboard identity, schemas, authoring, visualization,
variables, panels, file provisioning, Git Sync, cloud migration, and reporting.

## Dashboard identity and HTTP APIs

- In 11.5.0, Enterprise Analytics Views deprecates `:dashboardID` routes in
  favor of `uid/:dashboardUID`; Analytics Summaries likewise replaces
  `dashboard_id` routes with `dashboard_uid`.
- In 12.0.0, `/apis` dashboard endpoints perform fine-grained access-control
  checks. `kubernetesClientDashboardsFolders` is enabled by default.
- In 12.1.0, preferences identify the home dashboard by UID instead of numeric
  dashboard ID.
- In 12.2.0, deprecated internal-ID star APIs are removed.
- In 12.3.0, annotations saved using a dashboard UID no longer populate the
  internal numeric dashboard ID. Consumers must not require that field.
- In 12.3.0, dashboard requests from 12.3.2 enforce previously missing scope
  checks. API callers need the applicable scope.
- In 12.4.0, deprecated internal-ID dashboard endpoints are removed and
  `/api/dashboards/home` is deprecated.
- In 13.0.0, dashboard and folder resource APIs graduate to `v1`. Dashboard
  `v2` aligns `TransformationKind` and Dashboard Preferences, and API-server
  clients can set a preferred resource version.
- In 13.1.0, `DashboardDTO.isStarred` is removed. The mutation API gains
  annotation CRUD, and a panel screenshot API is available.

## Schemas, export, and as-code authoring

- In 12.0.0, Grafana adds dashboard-schema validation and a configurable limit
  on the series displayed by a panel, with an explicit render-all option.
- In 12.0.0, App Platform adds an experimental GitHub-backed dashboard
  configuration integration.
- In 12.1.0, exporting a schema V2 dashboard as `V1Resource` automatically
  transforms it to the compatible representation.
- In 12.3.0, Enterprise reporting supports schema V2 dashboards.
- In 12.4.0, dashboard provisioning supports schema V2. Provisioned dashboards
  can be edited through their JSON model, and schema-V2 report forms can edit
  template variables.
- In 13.0.0, the As Code editor includes schema validation; schema-V2 imports
  can carry labels. Authors can choose a default layout, create rows and tabs
  from the side pane, and define section-level variables.
- In 13.1.0, V1-to-V2 conversion preserves the timezone user preference and
  query-variable sort modes. File-defined V2 dashboards can be selected as the
  home dashboard.

## Git Sync and repository provisioning

- In 12.1.0, App Platform provisioning adds a pure-Git repository type and an
  experimental `nanogit` mode.
- In 12.2.0, Git Sync switches to inline secrets. Update repository
  provisioning configuration for this breaking format change.
- In 12.3.0, file provisioning watches the filesystem for changes instead of
  depending only on its initial scan.
- In 12.4.0, alert rules cannot be saved into Git-synced folders.
- For the 13.0-upgrade, do not use the withdrawn 13.0.0 as a stopping point
  when a self-managed 12.x instance uses the relevant Git Sync flags. Upgrade
  directly to 13.0.1 or later and use the pre-upgrade database restore strategy
  described in the upgrade reference.
- In 13.0.0, provisioning and Git Sync are enabled by default. Repository
  checks cover branch protection, write access, and emptiness. Git submodules
  are ignored, pure-Git URLs need not end in `.git`, and repository specs
  accept a custom webhook base URL.
- In 13.0.0, folder metadata is enabled by default. Exports create new UIDs;
  unmanaged resources cannot be overridden; repository-managed folders reject
  `ownerReferences` and manager-property changes. From 13.0.3, creating or
  moving a dashboard into a new folder writes `_folder.json`.
- In 13.1.0, repositories expose GPG, SSH, and S/MIME commit-signing settings.
  Repository identity is the combination of URL, branch, and path, and
  write-workflow checks respect ruleset bypasses.
- In 13.1.0, the webhook connector rejects GET, a provisioning URL change
  requires a fresh token, and webhook secrets rotate. GitHub webhooks use
  replay protection; files and history endpoints validate the `ref` query
  parameter.
- In 13.1.0, `public_root_url` controls externally visible provisioning URLs.
  From 13.1.1, the per-resource synchronization write timeout is configurable.

## Unified-storage migration and ownership

- During the 13.0-upgrade, the first startup copies folders and dashboards
  from legacy SQL tables to unified storage and records completion in
  `unifiedstorage_migration_log`. Legacy dashboard/folder tables cease to be
  authoritative. Restore the pre-upgrade database to roll back.
- Do not write new automation against the deprecated legacy tables. A downgrade
  reads stale data, and post-downgrade changes are not picked up on another
  upgrade after the one-time migration has been recorded.
- For persistent SQLite lock errors, raise
  `[unified_storage] migration_cache_size_kb` or enable
  `migration_parquet_buffer`.

## Dashboard controls, time, and variables

- In 11.6.0, dashboard models can define custom quick time ranges, and the time
  picker accepts manually specified quick ranges. Plugin/frontend `WeekStart`
  is typed as `WeekStart | undefined`, not an arbitrary string.
- In 11.6.0, time regions accept cron expressions.
- In 12.1.0, quick ranges can be configured at server level.
- In 12.2.0, variables can render below a drop-down. New-layout repeated panels
  support full-screen and solo-panel embed routes; repeating stops using clone
  keys. The Inspect drawer can no longer be opened or linked through a URL.
- In 12.3.0, dashboards add the `Switch` variable type. The controls menu
  exposes annotations; time-comparison windows can be saved; and view mode can
  change panel time-range settings.
- In 12.4.0, variable regular expressions can transform display text, and
  time-series dashboards support per-panel filtering.

## Panels, visualizations, and transformations

- In 11.5.0, **Extract fields** supports Delimiter and RegExp formats.
  Transformation filtering can select multiple query RefIDs. Trace-view span
  filters can be saved as panel options.
- In 11.6.0, variables work across all transformations; unary **Add field from
  calculation** adds `round()`. Histogram supports multiple native histograms.
- In 11.6.0, Canvas elements can execute one-click links and actions.
  Visualization actions can request confirmation before executing.
- In 12.0.0, `Stack` and `Grid` expose `columnGap` and `rowGap`.
- In 12.1.0, State timeline displays `false` and empty strings and permits
  mappings for `NaN` and `null`. Organize fields adds Auto mode; Regression is
  generally available; XY charts accept time on the x-axis.
- In 12.2.0, Canvas can disable tooltips for one-click elements and choose
  connection direction dynamically. Pie charts add ascending, descending, and
  disabled sorting.
- In 12.2.0, tables add frozen columns, maximum row height for variable-height
  rows, and field-derived tooltips. Transpose gets empty-value options;
  Trend/TimeSeries gets value labels; Trend supports a logarithmic x-axis.
- In 12.3.0, Canvas background images may come from non-icon fields. Time
  series panels accept custom x-axis time units.
- In 12.3.0, tables render array-valued `FieldType.other` values as pills,
  format Pill and JSON cells, and add links or actions to sparkline cells.
  Geomap supports MapLibre styles as base layers, and previously beta layers
  become generally available.
- In 12.4.0, click-and-drag time panning is generally available for time series
  and also works in candlestick, heatmap, and timeline panels. Heatmaps add a
  linear y-axis; Geomap XYZ tiles accept variables and min/max zoom; smoothing
  is available as a transformation.
- In 13.0.0, dashboard Logs panels can expose a field selector, persist shown
  fields, and hide the Level field.

## Library panels and reusable elements

- In 12.1.0, library-panel RBAC is generally available and enabled by default;
  `libraryPanelRBAC` is removed. Library elements can no longer be library
  variables.
- In 12.3.0, library-panel names cease to be unique. Use stable IDs.
- In 12.4.0, Library Elements deprecates `folderFilter` in favor of
  `folderFilterUIDs`.

## Cloud migration

- In 11.5.0, Cloud Migrations is enabled by default and the migration assistant
  has a dedicated RBAC role.
- In 12.1.0, mute timings are treated as notification-policy dependencies and
  migrate with the related policy resources.
- In 12.4.0, the Cloud Migrations feature toggle is removed. Use its
  configuration setting when the feature must be disabled.

## Reporting and rendering behavior

- In 11.5.0, Enterprise reporting adds an allowed-email-domain setting, uses
  the API server by default, and deprecates internal IDs.
- In 11.6.0, report emails can set their subject.
- In 12.4.0, report retries are productized, stabilized PDF rendering no longer
  uses `newPDFRendering`, and schema-V2 forms can edit template variables.
- In 13.0.0, PDF reports add header toggles, configurable footers, and a
  readiness observer.
- In 13.1.0, backend reporting can render by URL and can limit report-email
  recipients to organization members.

## Restored and removed dashboard features

- In 11.6.0, scripted dashboards return after their earlier removal.
- In 12.0.0, experimental dashboard restore behind `dashboardRestore` is
  removed.
- In 13.0.0, dashboard restore is enabled by default through the replacement
  implementation.
- In 13.1.0, `dashboardScene` and `publicDashboardsScene` are removed.
