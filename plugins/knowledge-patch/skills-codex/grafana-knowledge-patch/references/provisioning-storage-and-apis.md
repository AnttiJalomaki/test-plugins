# Provisioning, Storage, and HTTP APIs

Use this reference for dashboard and alert provisioning, Git Sync, resource
ownership, UID migrations, unified storage, webhooks, and the transition from
legacy HTTP endpoints to versioned resource APIs.

## UID-first API design

Prefer UIDs for every durable integration:

- Analytics Views moves from `:dashboardID` to `uid/:dashboardUID`, and
  Analytics Summaries moves from `dashboard_id` to `dashboard_uid` (since
  11.5.0).
- `failWrongDSUID` is enabled by default during the `12.0-upgrade`. Data-source
  writes reject invalid characters and UIDs longer than 40 characters.
- Home-dashboard preferences use dashboard UID, not numeric ID (since 12.1.0).
- Internal-ID star APIs are removed (since 12.2.0).
- Annotations written by dashboard UID do not store the numeric dashboard ID
  (since 12.3.0).
- `PUT /api/datasources/uid/:uid` requires the body UID to equal the path UID
  and otherwise returns HTTP 400 (since 12.3.0).
- Internal-ID dashboard routes are removed, `/api/dashboards/home` is
  deprecated, and name- or internal-ID-based data-source routes are deprecated
  (since 12.4.0).
- During the `13.0-upgrade`, numeric-ID data-source endpoints are disabled by
  default. Use UID routes; `datasourceLegacyIdApi` is only a temporary bridge.
- Usage Insights identifies dashboards and data sources by UID (since 13.0.0).
- External Alertmanager sender metrics identify data sources by UID (since
  13.1.0).

When strict UID validation finds an invalid value, create a new data source and
repoint dashboard JSON and alert-rule queries. Do not assume the UID can be
changed in place.

## Kubernetes-style API migration

`alertingApiServer` is enabled by default from 11.5.0. Notification-policy
trees and previews consume the Kubernetes API, and template-group responses
include built-in defaults.

Dashboard endpoints under `/apis` apply fine-grained authorization, while
`kubernetesClientDashboardsFolders` is enabled by default (since 12.0.0).
`kubernetesDashboards` becomes enabled by default in 12.2.0.

The legacy `/api` family is deprecated in the `13.0-upgrade` and remains
temporarily enabled. Build new automation against versioned `/apis` resources.
Dashboard and folder APIs graduate to `v1` in 13.0.0; dashboard `v2` aligns
`TransformationKind` and Dashboard Preferences. API-server clients can select
a preferred resource version.

## Alertmanager resource APIs

Grafana 12.0.0 removes manual editing and restoration of the internal
Alertmanager configuration and removes the internal POST write endpoint.

The `13.0-upgrade` removes:

- `DELETE /api/alertmanager/grafana/config/api/v1/alerts`
- `POST /api/alertmanager/grafana/config/api/v1/receivers/test`

It restricts these endpoints to administrators:

- `GET /api/alertmanager/grafana/config/api/v1/alerts`
- `GET /api/alertmanager/grafana/config/history`
- `POST /api/alertmanager/grafana/config/history/{id}/_activate`

Move notification automation below
`/apis/notifications.alerting.grafana.app/v1beta1/namespaces/{namespace}/`.
Use `receivers`, `routingtrees`, `templategroups`, `timeintervals`, and
`inhibitionrules` for contact points, policies, templates, mute timings, and
inhibition rules. Receiver testing already moves to Kubernetes-style App
Platform APIs in 12.4.0.

Grafana 13.0.0 provisioning accepts resource-specific permissions and enforces
protected-field authorization. Notification APIs enforce provenance. Managed
routes use `notification_settings.policy`, and legacy notification actions and
legacy alert-rule provisioning routes are deprecated. `alerting.rulesAPIV2`
and Rules API v2 begin the alert rule transition in 13.1.0; notification
provisioning endpoints are deprecated.

## File provisioning

- File provisioning accepts alert `keepFiringFor` and
  `missing_series_evals_to_resolve` (since 12.2.0).
- Provisioning watches the filesystem for changes rather than scanning only at
  startup (since 12.3.0).
- Dashboard provisioning supports schema V2, and provisioned dashboards can be
  edited through their JSON model (since 12.4.0).
- Alert rules cannot be saved into Git-synced folders (since 12.4.0).
- File-defined V2 dashboards can be selected as the home dashboard (since
  13.1.0).

## Git-backed provisioning

- The App Platform adds an experimental GitHub dashboard-configuration
  integration (since 12.0.0).
- Pure-Git repositories and experimental `nanogit` mode are available from
  12.1.0.
- Git Sync provisioning moves to inline secrets in 12.2.0. This is a breaking
  provisioning configuration change.

Provisioning and Git Sync are enabled by default in 13.0.0. Repositories are
checked for branch protection, write access, and emptiness. Git submodules are
ignored, pure-Git URLs no longer require a `.git` suffix, and repository
specifications can set a custom webhook base URL.

Folder metadata is enabled by default in 13.0.0. Exports generate new UIDs,
unmanaged resources cannot be overwritten, and repository-managed folders
reject `ownerReferences` and manager-property changes. From 13.0.3, creating
or moving a dashboard into a new folder also writes `_folder.json`.

## Grafana 13.0.0 Git Sync incident

Grafana 13.0.0 was withdrawn because a self-managed 12.x instance using
`provisioning`, `kubernetesClientDashboardsFolders`, `kubernetesDashboards`,
and `grafanaAPIServerEnsureKubectlAccess` can lose or revert dashboards and
folders. Upgrade directly to 13.0.1 or later.

For mixed local/Git content, or uncertain deployment mode, restore the
pre-upgrade database before proceeding. A deployment whose entire instance is
authoritative in Git may upgrade and resync. Installing a later version over an
affected 13.0.0 database does not itself restore content.

## Repository signing and identity

Grafana 13.1.0 exposes GPG, SSH, and S/MIME commit-signing configuration.
Repository uniqueness is the tuple of URL, branch, and path. Write-workflow
validation recognizes ruleset bypasses.

## Webhook security

Provisioning webhook behavior is hardened in 13.1.0:

- the connector rejects GET;
- changing the provisioning URL requires a new token;
- webhook secrets rotate periodically;
- GitHub webhooks use replay protection;
- files and history endpoints validate the `ref` query parameter.

`public_root_url` controls externally visible provisioning URLs. From 13.1.1,
the per-resource sync write timeout is configurable.

## Unified storage

- Unified Storage supports PostgreSQL `verify-full` and prefers TLS when the
  main database connection uses SSL (since 11.5.0).
- Storage honors `migration_locking`, and Unified Storage respects
  `GF_DATABASE_URL` (since 12.1.0).

On the first Grafana 13 start, folders and dashboards migrate from legacy SQL
tables to unified storage. Completion is stored in
`unifiedstorage_migration_log`. The former dashboard, ACL, provisioning,
version, tag, library connection, and folder tables are no longer
authoritative. A binary downgrade reads stale state; rollback requires
restoring the pre-upgrade database.

SQLite lock failures automatically retry through a Parquet buffer. If locking
persists, increase `[unified_storage] migration_cache_size_kb` beyond its
`1000000` default or set `migration_parquet_buffer = true`.

## Cloud Migration resources

Cloud Migrations is enabled by default and has a dedicated assistant role from
11.5.0. From 12.1.0, mute timings are dependencies of notification policies,
so migrations carry both. In 12.4.0, the feature toggle is replaced by a
configuration setting that can disable Cloud Migrations.

## Post-provisioning checks

After any provisioning or storage transition:

1. Compare dashboard and folder counts, UIDs, paths, resource versions, and
   ownership with the pre-change inventory.
2. Verify Git branch, path, signing, webhook secret, replay behavior, and
   `_folder.json` output.
3. Exercise filesystem watches, schema V2 edits and exports, home dashboards,
   protected fields, and provenance.
4. Exercise every migrated `/apis` resource with the intended role.
5. Confirm the database backup can restore authoritative state; do not treat a
   binary downgrade as rollback after unified-storage migration.
