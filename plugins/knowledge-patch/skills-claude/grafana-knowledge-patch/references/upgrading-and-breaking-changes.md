# Upgrading and breaking changes

Read this reference before changing major versions or carrying old
configuration, API clients, images, service commands, and feature flags into a
new deployment.

## Upgrade to 12.x

### Audit data-source UIDs

During the 12.0-upgrade, `failWrongDSUID` becomes enabled by default. REST and
provisioning create/update requests reject UIDs longer than 40 characters or
containing characters other than letters, digits, hyphens, and underscores.
Audit the existing data sources, create replacements for invalid UIDs, and
repoint dashboard JSON and alert queries:

```bash
curl http://localhost:3000/api/datasources |
  jq '.[] | select((.uid | test("^[a-zA-Z0-9\\-_]+$") | not) or (.uid | length > 40)) | {id, uid, name, type}'
```

Add authentication when required.

### Reserve migration disk

The 12.0-upgrade fills `annotation.dashboard_uid` by rewriting the full
`annotation` table and rebuilding indexes. Before an 11.x-to-12.x upgrade,
back up the database and reserve at least two to three times the table size in
free space. A failed migration can prevent startup.

After a successful upgrade, reclaim space in a low-traffic maintenance window;
these operations lock the table:

```sql
-- PostgreSQL
VACUUM FULL annotation;

-- MySQL
OPTIMIZE TABLE annotation;

-- SQLite
VACUUM;
```

### Plugin compatibility

During the 12.0-upgrade, `grafana cli plugins install` enforces each plugin's
`grafanaDependency` and rejects versions incompatible with the running Grafana.
There is no bypass flag; a deliberately incompatible plugin requires the ZIP
installation path.

In 12.0.0, Angular frontend support is removed. Plugin dependency-version
support and secrets-manager plugins are also removed. Inventory and update
plugins before upgrading.

## Upgrade to 13.x

### Avoid 13.0.0 for affected Git Sync installations

The 13.0-upgrade guidance is to upgrade directly to 13.0.1 or later. The
withdrawn 13.0.0 can lose or revert dashboards and folders on self-managed 12.x
instances using the `provisioning`, `kubernetesClientDashboardsFolders`,
`kubernetesDashboards`, and `grafanaAPIServerEnsureKubectlAccess` Git Sync
flags.

For mixed local and Git Sync content, or an uncertain deployment mode, restore
the pre-upgrade database before moving to 13.0.1. A full-instance Git Sync
deployment can upgrade and resync from Git. Upgrading an already affected
13.0.0 database does not itself restore content.

### Sequence React 19 and plugin work

Grafana 13 uses React 19. During the 13.0-upgrade, first move the current
Grafana line to its latest patch, then update and validate all installed
plugins, and only then upgrade Grafana.

### Plan the one-way unified-storage migration

On first 13.0 startup, folders and dashboards migrate from legacy SQL storage
to unified storage, and completion is written to
`unifiedstorage_migration_log`. These tables then become deprecated and
non-authoritative:

- `dashboard`
- `dashboard_acl`
- `dashboard_provisioning`
- `dashboard_version`
- `dashboard_tag`
- `library_element_connection`
- `folder`

A downgrade reads stale legacy tables. Restore the pre-upgrade backup to roll
back. Changes made after a downgrade are not picked up by a later upgrade
because the one-time migration remains recorded.

SQLite automatically retries lock errors with a Parquet buffer. If
`database is locked` persists, raise
`[unified_storage] migration_cache_size_kb` above `1000000` or explicitly set:

```ini
[unified_storage]
migration_parquet_buffer = true
```

### Migrate commands and renderer

The 13.0-upgrade removes the `grafana-cli` and `grafana-server` command names.
Update units, containers, CI, and scripts to use `grafana cli` and
`grafana server`.

Plugin-mode Image Renderer is also removed. Run it as a separate service.
`renderAuthJWT` is enabled by default; set the same nonempty and non-`-`
secret in Grafana and the renderer, then restart Grafana:

```ini
[rendering]
renderer_token = replace-with-a-shared-secret
```

The former database-backed opaque-token behavior can be restored temporarily:

```ini
[feature_toggles]
renderAuthJWT = false
```

### Move HTTP clients toward UIDs and `/apis`

During the 13.0-upgrade, the legacy `/api` family becomes deprecated in favor
of versioned Kubernetes-style `/apis` resources. Numeric-ID data-source routes
are disabled by default. Use UIDs; `datasourceLegacyIdApi` can temporarily
re-enable the old routes until both routes and flag are removed.

## Defaults that change behavior

- In 11.5.0, `alertingApiServer` is enabled by default; Cloud Migrations is
  enabled by default; alert retry `max_attempts` defaults to 3; and Loki label
  lookup changes from `/series` to `/labels` plus a `query` parameter.
- In 11.6.0, existing API keys migrate to service accounts during startup.
- In 12.0.0, `failWrongDSUID`,
  `kubernetesClientDashboardsFolders`, and
  `prometheusRunQueriesInParallel` are enabled by default. Anonymous users are
  constrained to their Viewer organization role, and standard datetime units
  are limited to millisecond precision.
- In 12.1.0, recording rules, improved OAuth/SAML session handling, and
  `ssoSettingsLDAP` are enabled by default.
- In 12.2.0, `alertingSaveStateCompressed` and `kubernetesDashboards` are
  enabled by default.
- In 12.4.0, short URLs never expire by default, the new Logs visualization is
  enabled by default, native HTTP histograms are the default, and plugin
  processes stop inheriting the host environment.
- In 13.0.0, gzip is enabled by default through `server.enable_gzip`;
  provisioning, Git Sync, folder metadata, and dashboard restore are enabled
  by default. Disable gzip explicitly if another layer owns compression.
- In 13.1.0, no listed default change replaces these; review removed gates and
  API migrations instead.

## Removed and transitioned feature flags

Remove obsolete flag names from configuration. A feature becoming generally
available or default does not mean its old flag remains accepted.

- In 11.5.0, `newFiltersUI` becomes generally available;
  `cloudwatchMetricInsightsCrossAccount` and `publicDashboards` are removed.
- In 11.6.0, `alertingNoNormalState`, `sqlQuerybuilderFunctionParameters`,
  `openSearchBackendFlowEnabled`, `managedPluginsInstall`, and
  `accessControlOnCall` are removed.
- In 12.0.0, `alertingNoDataErrorExecution`, Loki Alert State History gates,
  `queryOverLive`, `live-service-web-worker`, `userStorageAPI`,
  `traceQLStreaming`, and experimental `dashboardRestore` are removed.
  `pluginsSriChecks` is generally available.
- In 12.1.0, `libraryPanelRBAC` is removed because the feature is generally
  available. The experimental Loki query-splitting configuration and
  predefined operations are removed.
- In 12.2.0, `prometheusCodeModeMetricNamesSearch`,
  `HideAngularDeprec`, and the nested-folders flag are removed.
- In 12.3.0, the deprecated experimental API-server toggle is removed.
- In 12.4.0, `logRequestsInstrumentedAsUnknown`, `pinNavItems`,
  `unifiedHistory`, `individualCookiePreferences`,
  `permissionsFilterRemoveSubquery`, `logRowsPopoverMenu`,
  `logsInfiniteScrolling`, `exploreMetricsRelatedLogs`, and
  `postgresDSUsePGX` are removed. Drilldown Investigations and CSV
  drag-and-drop snapshot queries are removed.
- In 12.4.0, `localeFormatPreference`, Datagrid,
  `GrafanaBootData.config.apps`, `GrafanaBootData.config.panels`, and
  `getFolderByUID` are deprecated. Library Elements moves from `folderFilter`
  to `folderFilterUIDs`; the Faro v2 upgrade removes
  `web_vitals_attribution_enabled`.
- In 13.0.0, environment variables can configure feature toggles directly;
  `GF_FEATURE_TOGGLES_ENABLE` and `[feature_toggles] enable` are deprecated.
  `newFiltersUI` and `kubernetesAlertingRules` are removed.
- In 13.1.0, `alertRuleUseFiredAtForStartsAt`, `dashboardScene`,
  `publicDashboardsScene`, and `logsPanelControls` are removed.

## Removed and deprecated HTTP surfaces

- In 12.0.0, internal Alertmanager configuration writes are removed, including
  its internal POST endpoint. `viewers_can_edit` is removed.
- In 12.2.0, internal-ID star APIs are removed.
- In 12.4.0, internal-ID dashboard routes are removed,
  `/api/dashboards/home` is deprecated, and data-source routes based on names
  or internal IDs are deprecated in favor of UIDs.
- During the 13.0-upgrade, legacy Alertmanager DELETE and receiver-test POST
  endpoints are removed; several GET/history endpoints become admin-only.
- In 13.0.0, Enterprise removes `/access-control/assignments/search` and
  `IncludeMapped` from the user-role listing API.
- In 13.1.0, notification provisioning endpoints, GroupAttributeSync routes,
  and the dashboard-version metric are removed. `DashboardDTO.isStarred` is
  removed.

## Container and base-image transitions

- In 11.5.0, the frontend toolchain moves to Node 22.
- In 11.6.0, Docker images use Grafana-provided glibc 2.40 binaries.
- In 12.2.0, migrate image references from deprecated
  `grafana/grafana-oss` to `grafana/grafana`.
- From 12.3.5, Image Renderer supports custom CAs for privately trusted TLS.
  From 12.3.8, Alpine images use Alpine 3.24.1.
- In 13.0.0, the Ubuntu base moves from 22.04 to 24.04.

Rebuild and test derived images whenever the base distribution or libc changes.

## Post-upgrade verification

1. Confirm migrations completed and that the authoritative storage contains the
   expected dashboards, folders, annotations, alert rules, and permissions.
2. Search startup logs for removed settings, feature flags, or malformed UIDs.
3. Exercise both read and write paths used by API and provisioning clients.
4. Test all installed plugins, external renderer calls, data-source backend
   connectivity, authentication flows, custom roles, and alert delivery.
5. Compare dashboards and metric selectors with renamed or removed metrics.
