---
name: grafana-knowledge-patch
description: Grafana
version: 13.1.0
license: MIT
metadata:
  author: Nevaberry
---


# Grafana Knowledge Patch

Use this skill when upgrading, configuring, extending, or integrating with
Grafana and the work may depend on recent API, alerting, dashboard, data-source,
authentication, plugin, or deployment behavior.

## How to use this skill

1. Determine the running Grafana version and edition from the deployment,
   manifest, image tag, or server build information.
2. Read the upgrade and breaking-change reference before changing a major
   version.
3. Open the task-specific reference from the index below.
4. Treat manifests, configuration, code, and observed behavior as authoritative
   when they disagree with compatibility guidance.
5. Gate Enterprise-only and preview features explicitly; do not assume that a
   feature available in one edition or deployment mode exists in another.
6. Prefer UID-based and versioned Kubernetes-style APIs for new integrations.
7. Test provisioning, RBAC, alert delivery, data-source queries, rendering, and
   plugins after an upgrade rather than relying only on a successful startup.

## Reference index

| Reference | Topics |
| --- | --- |
| [upgrading-and-breaking-changes.md](references/upgrading-and-breaking-changes.md) | Major-upgrade runbooks, migrations, removed flags, commands, APIs, images, and defaults |
| [alerting.md](references/alerting.md) | Rules, evaluation, state, recording rules, Alertmanager, contact points, templates, imports, and HA |
| [dashboards-and-provisioning.md](references/dashboards-and-provisioning.md) | Dashboard APIs and schema, panels, transformations, variables, Git Sync, file provisioning, reporting, and migration |
| [data-sources-and-observability.md](references/data-sources-and-observability.md) | Prometheus, Loki, Tempo, Elasticsearch, CloudWatch, SQL, traces, profiles, logs, expressions, and metrics |
| [authentication-and-access.md](references/authentication-and-access.md) | OAuth, SAML, JWT, SCIM, service accounts, RBAC, permissions, identities, and audit controls |
| [plugins-frontend-and-runtime.md](references/plugins-frontend-and-runtime.md) | Plugin compatibility, frontend APIs, extensions, renderer, sandboxing, server runtime, containers, and process environment |

## Upgrade first: high-risk changes

### Do not stop on Grafana 13.0.0 with affected Git Sync deployments

Grafana 13.0.0 was withdrawn because an affected self-managed Git Sync upgrade
can lose or revert dashboards and folders. Upgrade directly from the latest
12.x patch to 13.0.1 or later. If local and Git-managed content were mixed, or
the deployment mode is uncertain, restore the pre-upgrade database before
retrying. A full-instance Git Sync deployment may resync from Git, but merely
upgrading an affected 13.0.0 database does not recover lost content.

### Treat unified-storage migration as one-way

The first 13.0 startup migrates dashboards and folders from legacy SQL tables
to unified storage and records completion in `unifiedstorage_migration_log`.
Afterward, legacy dashboard and folder tables are not authoritative. Rollback
requires the pre-upgrade database backup; a downgrade reads stale tables.

For repeated SQLite lock failures, increase
`[unified_storage] migration_cache_size_kb` above its `1000000` default or
stage through Parquet:

```ini
[unified_storage]
migration_parquet_buffer = true
```

### Reserve space for the 12.0 annotation migration

The 11.x-to-12.x migration rewrites the complete `annotation` table and rebuilds
its indexes to populate `annotation.dashboard_uid`. Back up the database and
reserve at least two to three times the table size in free space. Reclaim space
later in a low-traffic window because `VACUUM FULL`, `OPTIMIZE TABLE`, and
SQLite `VACUUM` lock the table.

### Audit data-source UIDs before 12.0

Malformed data-source UIDs are rejected by REST and provisioning when
`failWrongDSUID` is enabled by default. Valid UIDs contain only letters,
digits, hyphens, and underscores and are at most 40 characters. Replace invalid
data sources, then repoint dashboard JSON and alert-rule queries.

```bash
curl http://localhost:3000/api/datasources |
  jq '.[] | select((.uid | test("^[a-zA-Z0-9\\-_]+$") | not) or (.uid | length > 40)) | {id, uid, name, type}'
```

Add authentication to the audit request when the API requires it.

### Prepare plugins before a 13.x upgrade

Grafana 13 uses React 19. First update the current Grafana release line to its
latest patch, then update and validate every installed plugin, and only then
upgrade Grafana. Angular frontend support is already absent in 12.x, and
plugin-mode Image Renderer is absent in 13.x.

## API migration rules

- Use stable UIDs rather than numeric IDs or display names for dashboards,
  folders, data sources, annotations, library panels, analytics, and usage
  integrations.
- Migrate new automation from the legacy `/api` family toward versioned
  `/apis` resources. In 13.0, numeric-ID data-source endpoints are disabled by
  default; `datasourceLegacyIdApi` is only a temporary bridge.
- Keep the payload UID identical to the path UID for
  `PUT /api/datasources/uid/:uid`; mismatches return HTTP 400.
- Do not depend on `DashboardDTO.isStarred`, internal dashboard IDs on
  UID-saved annotations, star APIs, or removed dashboard-version metrics.
- For Alertmanager configuration, move automation to the App Platform
  notification resources. Legacy configuration write and test endpoints are
  removed or restricted.
- Expect `/apis` dashboard operations to enforce fine-grained scopes and
  protected-field or provenance permissions.

## Alerting quick reference

- Recording rules are enabled by default from 12.1. Set
  `default_datasource_uid`, choose per-rule write targets when needed, and
  verify PDC and per-data-source write settings.
- Use rule UIDs as identity. Rule titles and library-panel names are no longer
  unique, and rules may exist without a group.
- Model recovery with `keep_firing_for` and `Recovering`; configure
  `missing_series_evals_to_resolve` where missing series should resolve only
  after multiple evaluations.
- File provisioning accepts `keepFiringFor` and
  `missing_series_evals_to_resolve`. Provisioned recording rules may omit a
  condition, but receivers with an empty name are invalid.
- Compressed alert-state persistence is the default from 12.2. State storage
  supports batching, jitter, retry with backoff, and configurable save
  behavior.
- Notification receivers support HMAC webhook signatures, templatable
  payloads, OAuth2, external Alertmanager mTLS, Jira, Slack colors, and
  time-range-aware dashboard and panel links.
- In 13.0, managed route assignment uses
  `notification_settings.policy`, not labels. Provisioning and notification
  APIs enforce more granular authorization and provenance.
- In 13.1, notification provisioning endpoints are deprecated and Rules API v2
  is available behind `alerting.rulesAPIV2`.

## Dashboard and provisioning quick reference

- Prefer schema V2 for new as-code work where supported. Provisioning, export,
  reporting, conversion, and home-dashboard handling have specific V2 paths;
  validate round trips.
- Provisioning and Git Sync are enabled by default in 13.0. Repository checks
  cover branch protection, write access, and emptiness, and managed resources
  have stricter ownership rules.
- Git Sync uses inline secrets from 12.2. Webhooks reject GET, secrets rotate,
  URL changes require a new token, and GitHub deliveries have replay
  protection.
- File provisioning watches for changes, and file-defined V2 dashboards may be
  selected as the home dashboard.
- Dashboard preferences and analytics should reference dashboard UIDs.
- Use the As Code editor, schema validation, labels on V2 imports, rows, tabs,
  section variables, and default layouts where those authoring features fit.

## Data-source and query quick reference

- Loki label lookup defaults to `/labels` with a `query` parameter rather than
  `/series`; update proxies, permissions, and integrations that assume the old
  endpoint.
- Elasticsearch field discovery uses `_field_caps`, so proxies and permissions
  must allow it. Core Elasticsearch is no longer bundled in 13.0, even though
  the separately installed integration adds ES|QL and variable queries.
- Core Prometheus no longer includes Azure or SigV4 authentication in 13.1, and
  the `grafana-prometheus` package is removed.
- Zipkin moves to backend-routed queries and is later removed from core in
  13.1; server-side network access and explicit installation matter.
- Tempo streaming header forwarding was disabled in 12.4 and restored in 13.0.
  Verify the exact running version before relying on incoming or team headers.
- Prometheus queries containing `$__range` bypass incremental querying.
- Server-side expression pipelines can return unaffected partial results when
  one node fails; SQL expressions support `NOT`, CTEs in alerts, variable
  interpolation, and public-preview workflows.

## Authentication and authorization quick reference

- Existing API keys migrate to service accounts on first 11.6 startup; API-key
  endpoints and authentication code are removed in 12.1.
- OAuth supports `client_secret_jwt`, ID-token signature checks, required
  refresh tokens, access-token user-info extraction, and workload identity.
  Exhausted token-refresh retries now surface as errors.
- JWT supports organization-role mapping, TLS controls, bearer-token files,
  client CAs, and inline public keys.
- Enterprise data-source queries require the `query` permission; `read` is not
  an alternative. Drilldown requires `datasources:explore`.
- In 13.0, audit logging omits data-source request and response bodies by
  default. Opt in only when their diagnostic value justifies the exposure.
- Custom roles must not retain deprecated annotation permissions or global
  data-source UID scopes. Use dashboard/folder permissions for dashboard
  annotations and non-global roles for UID-scoped data-source access.

## Plugin and deployment quick reference

- `grafana cli plugins install` enforces `grafanaDependency`; incompatible
  versions require the ZIP path because no bypass flag exists.
- Plugin processes no longer inherit host environment variables by default.
  Pass required values deliberately; restricted setups receive
  `PLUGIN_UNIX_SOCKET_DIR`, and external AWS plugins retain AWS credential-chain
  variables.
- Every `plugin.json` `routes[]` entry needs `path`, and every `includes[]`
  entry needs `type`.
- Replace deprecated `Select` with `Combobox`; account for the later
  `isItemDisabled` rename and removed `Gauge` component.
- Use the observable and asynchronous plugin/data-source APIs instead of
  removed extension APIs or `datasourceSrv`.
- Run Image Renderer as a separate service in 13.x. Configure the same
  nonempty, non-`-` `[rendering] renderer_token` in Grafana and renderer; JWT
  rendering authentication is enabled by default.
- Invoke `grafana cli` and `grafana server`; the hyphenated legacy command
  names are removed.

## Validation checklist

- Back up the database and prove the restore procedure before a major upgrade.
- Check free disk, migration logs, database locks, and startup logs.
- Inventory feature toggles and settings; remove obsolete gates instead of
  carrying them forward.
- Exercise dashboard and folder CRUD through the API family used by automation.
- Validate Git Sync ownership, webhook, signing, branch, and write rules.
- Test alert evaluation, state persistence, routing, templates, remote writes,
  contact points, and Alertmanager synchronization.
- Test every data source from the server side, including TLS, proxy, header,
  identity-forwarding, and backend-routed requests.
- Re-evaluate custom roles and service-account permissions against the exact
  actions now enforced.
- Upgrade, sign-check, and smoke-test all plugins and renderer connections.
- Compare runtime assumptions in derived container images with the new base
  image and libc versions.
