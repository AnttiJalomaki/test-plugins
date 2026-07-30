# Upgrades, Runtime, and Removed Configuration

Use this reference for release transitions, host and container assumptions,
renderer deployment, command changes, and configuration cleanup.

## Major-upgrade runbooks

### Upgrade to Grafana 12

Before moving from 11.x, account for the `12.0-upgrade` migration work:

- `failWrongDSUID` is enabled by default. REST and provisioning writes reject
  malformed data-source UIDs, including values longer than 40 characters.
  Create a replacement data source for each invalid UID and update dashboard
  JSON and alert-rule queries; do not expect an in-place UID rename.
- `grafana cli plugins install` evaluates the plugin's `grafanaDependency`.
  There is no compatibility-bypass switch. A deliberate incompatible install
  must use the ZIP installation path.
- Populating `annotation.dashboard_uid` rewrites the complete annotation table
  and its indexes. Back up first and reserve at least two to three times the
  current table size. Insufficient space can prevent Grafana from starting.
- Reclaim disk only in a low-traffic maintenance window: `VACUUM FULL
  annotation` on PostgreSQL, `OPTIMIZE TABLE annotation` on MySQL, or `VACUUM`
  on SQLite. These operations lock the table.

Grafana 12.0 removes Angular frontend support, plugin dependency-version
support, secrets-manager plugins, internal Alertmanager configuration writes,
and `viewers_can_edit`. Anonymous access now enforces the configured Viewer
organization role. Dashboard schema validation and a configurable panel-series
cap also take effect (since 12.0.0).

### Upgrade to Grafana 13

The `13.0-upgrade` path has several hard guardrails:

- Do not deploy the withdrawn 13.0.0 to a self-managed 12.x Git Sync instance
  using `provisioning`, `kubernetesClientDashboardsFolders`,
  `kubernetesDashboards`, and `grafanaAPIServerEnsureKubectlAccess`. Upgrade
  directly to 13.0.1 or later.
- If 13.0.0 already touched mixed local and Git Sync content, or the mode is
  uncertain, restore the pre-upgrade database before installing 13.0.1. A
  full-instance Git Sync deployment may upgrade and resync from Git. Upgrading
  an affected database alone cannot recover lost content.
- Grafana 13 uses React 19. First update the current Grafana line to its latest
  patch, update and validate every plugin, and only then cross the major
  boundary.
- The legacy `/api` family is deprecated in favor of versioned
  Kubernetes-style `/apis`. Numeric-ID data-source endpoints are disabled by
  default. Use UIDs; `datasourceLegacyIdApi` is temporary.

The first Grafana 13 start performs a one-way folders-and-dashboards migration
to unified storage and records it in `unifiedstorage_migration_log`.
`dashboard`, `dashboard_acl`, `dashboard_provisioning`, `dashboard_version`,
`dashboard_tag`, `library_element_connection`, and `folder` are then
deprecated and non-authoritative. Downgrading reads stale legacy tables.
Rollback requires the pre-upgrade database backup, and changes written after a
downgrade are not picked up by another upgrade because migration completion is
already recorded.

SQLite retries lock failures with a Parquet buffer. For persistent `database
is locked` failures, raise `[unified_storage] migration_cache_size_kb` above
the `1000000` default or set:

```ini
[unified_storage]
migration_parquet_buffer = true
```

### Renderer transition

Image Renderer plugin mode gained SSL support in 11.6.0 and custom-CA support
in 12.3.5, but is removed by `13.0-upgrade`. Deploy the renderer as a separate
service. `renderAuthJWT` is enabled by default; set the same nonempty,
non-`-` `[rendering] renderer_token` in both services and restart Grafana.
Temporarily setting `renderAuthJWT = false` restores the former
database-backed opaque-token flow.

## Command, image, and dependency changes

- Grafana's frontend toolchain moves to Node 22 (since 11.5.0).
- Grafana Docker images use Grafana-provided glibc 2.40 binaries (since
  11.6.0); custom images and native plugins must not assume the older libc.
- `grafana/grafana-oss` is deprecated in favor of `grafana/grafana` (since
  12.2.0). Update image references.
- Alpine-based images use Alpine 3.24.1 from 12.3.8. Revalidate packages and
  runtime assumptions in derived images.
- Grafana 13.0.0 moves its Ubuntu base from 22.04 to 24.04. Revalidate any
  derived Ubuntu image.
- Deprecated `grafana-cli` and `grafana-server` commands are removed by the
  `13.0-upgrade`; use `grafana cli` and `grafana server` in units, containers,
  scripts, and CI.
- The plugin `update-all` operation no longer performs a separate uninstall
  step (since 11.6.0). Automation must not depend on that intermediate state.

## Server and transport behavior

- Unified Storage supports PostgreSQL `verify-full` and prefers TLS when the
  Grafana database connection uses SSL (since 11.5.0).
- Storage honors `migration_locking`, and Unified Storage honors the
  `GF_DATABASE_URL` override (since 12.1.0).
- Short URLs default to never expiring (since 12.4.0).
- Grafana's HTTP metrics use native histograms by default in 12.4.0; classic
  histograms remain configurable.
- Grafana Live adds `client_queue_max_size` (since 12.4.0).
- `server.enable_gzip` defaults to `true` (since 13.0.0). Explicitly disable it
  when compression belongs at a reverse proxy.
- Grafana can serve HTTPS and HTTP/2 over a Unix domain socket and can listen
  on TCP and Unix sockets simultaneously (since 13.0.0). Redis remote cache
  also supports `network=unix`.
- Pushing data to Grafana Live requires RBAC authorization (since 13.0.0).

## Feature-toggle cleanup

Remove toggles when their behavior is promoted, enabled by default, or removed:

- In 11.5.0, `newFiltersUI` is generally available;
  `cloudwatchMetricInsightsCrossAccount` and `publicDashboards` are removed.
- In 11.6.0, remove `alertingNoNormalState`,
  `sqlQuerybuilderFunctionParameters`, `openSearchBackendFlowEnabled`,
  `managedPluginsInstall`, and `accessControlOnCall`.
- In 12.0.0, remove `alertingNoDataErrorExecution`, the Loki Alert State
  History toggles, `queryOverLive`, `live-service-web-worker`,
  `userStorageAPI`, `traceQLStreaming`, and experimental `dashboardRestore`.
  `pluginsSriChecks` is generally available.
- In 12.1.0, remove `libraryPanelRBAC`; the behavior is generally available
  and enabled by default.
- In 12.2.0, remove `prometheusCodeModeMetricNamesSearch`,
  `HideAngularDeprec`, and the nested-folders flag. `kubernetesDashboards` is
  enabled by default.
- In 12.3.0, delete the deprecated experimental API-server toggle.
- In 12.4.0, remove `logRequestsInstrumentedAsUnknown`, `pinNavItems`,
  `unifiedHistory`, `individualCookiePreferences`,
  `permissionsFilterRemoveSubquery`, `logRowsPopoverMenu`,
  `logsInfiniteScrolling`, `exploreMetricsRelatedLogs`, and
  `postgresDSUsePGX`. Drilldown Investigations and CSV drag-and-drop snapshot
  queries are removed.
- In 12.4.0, `localeFormatPreference` is deprecated. Cloud Migrations moves
  from a toggle to a configuration setting, and stabilized PDF rendering no
  longer uses `newPDFRendering`.
- In 13.0.0, configure feature toggles directly through environment variables.
  `GF_FEATURE_TOGGLES_ENABLE` and `[feature_toggles] enable` are deprecated;
  `newFiltersUI` and `kubernetesAlertingRules` are removed. Restore dashboards
  is enabled by default.
- In 13.1.0, remove `alertRuleUseFiredAtForStartsAt`, `dashboardScene`,
  `publicDashboardsScene`, and `logsPanelControls`.

## Removed and deprecated surface

- Scripted dashboards returned in 11.6.0 after their earlier removal.
- The Datagrid panel, `GrafanaBootData.config.apps`,
  `GrafanaBootData.config.panels`, and `getFolderByUID` are deprecated (since
  12.4.0).
- Library Elements deprecates `folderFilter` in favor of
  `folderFilterUIDs`, and the Faro v2 upgrade removes
  `web_vitals_attribution_enabled` (since 12.4.0).
- Grafana 13.0.0 no longer bundles Prometheus dashboards. Enterprise query
  caching removes duplicate `grafana_caching_items` and
  `grafana_caching_size` metrics. Update metric consumers and dashboards.
- GroupAttributeSync routes and the dashboard-version metric are removed
  (since 13.1.0).
