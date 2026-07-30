---
name: grafana-knowledge-patch
description: Grafana
version: 13.1.0
license: MIT
metadata:
  author: Nevaberry
---


# Grafana Knowledge Patch

Use this skill when upgrading, configuring, provisioning, extending, or
troubleshooting modern Grafana. Start with the installed Grafana version and
deployment mode, then apply only advice whose version attribution matches the
deployment. Treat manifests, configuration, API responses, and tests from the
actual installation as authoritative.

## Reference index

| Reference | Topics |
| --- | --- |
| [upgrades-and-runtime.md](references/upgrades-and-runtime.md) | Major-upgrade runbooks, storage migrations, containers, commands, renderer, server defaults |
| [alerting.md](references/alerting.md) | Rules, evaluation, notification APIs, contact points, recording rules, state, HA |
| [dashboards-and-visualizations.md](references/dashboards-and-visualizations.md) | Dashboard APIs, variables, panels, transformations, layouts, logs, reporting |
| [identity-access-and-enterprise.md](references/identity-access-and-enterprise.md) | Authentication, SSO, SCIM, RBAC, audit, migration, reporting controls |
| [data-sources-and-observability.md](references/data-sources-and-observability.md) | Prometheus, Loki, Tempo, cloud and SQL sources, traces, profiles, query behavior |
| [plugins-and-frontend.md](references/plugins-and-frontend.md) | Plugin compatibility, manifests, process isolation, React and UI API transitions |
| [provisioning-storage-and-apis.md](references/provisioning-storage-and-apis.md) | Git Sync, Kubernetes-style APIs, UID migrations, storage, webhooks, resource ownership |

## Upgrade triage

1. Record the exact Grafana version, edition, database, image, installed
   plugins, provisioning mode, and active feature toggles.
2. Back up the database and Git-provisioned content before a major upgrade.
3. Audit numeric-ID API calls, deprecated command names, Angular plugins,
   Image Renderer deployment mode, and plugin environment-variable
   dependencies.
4. Update the current release line and every plugin before crossing the React
   19 boundary.
5. Reserve migration disk space, test rollback from backup, and validate alert,
   dashboard, provisioning, authentication, and reporting paths in staging.

## Critical 13.0 upgrade guardrails

Do not upgrade a self-managed Git Sync installation to Grafana 13.0.0. That
withdrawn release can lose or revert dashboards and folders when the relevant
provisioning and Kubernetes dashboard flags are in use. Upgrade directly to
13.0.1 or later. If 13.0.0 already touched a mixed local/Git deployment,
restore the pre-upgrade database before proceeding; merely installing a later
binary does not recover the lost content.

Grafana 13 performs a one-way first-start migration of folders and dashboards
from legacy SQL tables to unified storage. The old tables cease to be
authoritative, and downgrading exposes stale data. Rollback means restoring the
pre-upgrade database, not simply reinstalling the old binary.

If SQLite repeatedly reports `database is locked`, increase
`[unified_storage] migration_cache_size_kb` above its `1000000` default or
stage through Parquet:

```ini
[unified_storage]
migration_parquet_buffer = true
```

Replace removed command names in service units, images, and automation:

```text
grafana-cli     -> grafana cli
grafana-server  -> grafana server
```

Plugin-mode Image Renderer is removed. Run the renderer as a separate service
and configure the same nonempty, non-`-` shared token on both sides:

```ini
[rendering]
renderer_token = replace-with-a-shared-secret
```

The JWT renderer path is enabled by default. `renderAuthJWT = false` is only a
temporary compatibility escape hatch for the former opaque-token behavior.

## Grafana 12 preflight

The 11.x-to-12.x annotation migration rewrites the full `annotation` table and
rebuilds its indexes. Back up the database and reserve two to three times the
table size as free space. Reclaim space later in a low-traffic window because
the database-specific operations lock the table.

Grafana 12 enables strict data-source UID validation by default. UIDs must
match the accepted character set and be at most 40 characters. Invalid UIDs
cannot be repaired in place: create replacement data sources, then repoint
dashboard JSON and alert queries. Add authentication to this local audit when
required:

```bash
curl http://localhost:3000/api/datasources |
  jq '.[] | select((.uid | test("^[a-zA-Z0-9\\-_]+$") | not) or (.uid | length > 40)) | {id, uid, name, type}'
```

After the migration, reclaim annotation-table space with the appropriate
maintenance command:

```sql
-- PostgreSQL
VACUUM FULL annotation;

-- MySQL
OPTIMIZE TABLE annotation;

-- SQLite
VACUUM;
```

Grafana 12 also removes Angular frontend support, internal Alertmanager
configuration writes, `viewers_can_edit`, plugin dependency-version support,
and secrets-manager plugins. Review integrations before the first start.

## API and identity rules

Prefer stable UIDs over numeric IDs, names, or titles:

- Dashboard, annotation, analytics, home-dashboard, data-source, external
  Alertmanager metrics, and Usage Insights paths have progressively moved to
  UIDs.
- Rule and library-panel names are no longer guaranteed unique; use their
  stable identifiers.
- From Grafana 13, numeric-ID data-source endpoints are disabled by default.
  `datasourceLegacyIdApi` is temporary only.
- The legacy `/api` family remains deprecated; new automation should target
  versioned Kubernetes-style `/apis` resources.

Existing API keys migrate to service accounts on first 11.6 startup, and the
API-key endpoints and authentication implementation are removed in 12.1.
Inventory old clients before either transition.

## Alerting default changes

Check defaults rather than carrying old assumptions forward:

- Alert retry `max_attempts` became `3`, with
  `state_periodic_save_batch_size` available for save batching.
- Recording rules became enabled by default in 12.1.
- Compressed alert-state persistence became enabled by default in 12.2.
- Pending periods apply to NoData and Error states in 12.4.
- Grafana 13 managed routes use `notification_settings.policy`, not labels.

Use rule UIDs, not titles. Titles are not unique, rules may be groupless, and
multiple lifecycle and version-restore paths now exist. When provisioning,
preserve provenance and protected-field authorization.

The old internal Alertmanager configuration and notification-provisioning
routes are being removed. Move receiver, routing tree, template group, time
interval, and inhibition-rule automation to the notification resources under
`/apis/notifications.alerting.grafana.app/`.

## Plugin compatibility checklist

- Upgrade to the latest patch in the current Grafana line and update all
  plugins before moving to Grafana 13 and React 19.
- The frontend toolchain uses Node 22 from 11.5.
- The plugin CLI enforces `grafanaDependency`; incompatible installations
  require the deliberate ZIP path rather than a bypass flag.
- Grafana 13 requires `type` on every `plugin.json` `includes` entry.
- Plugin manifests require `routes[].path` from 12.4.
- Host environment variables are no longer inherited by plugin processes by
  default. Pass explicit configuration; use `PLUGIN_UNIX_SOCKET_DIR` for
  constrained temporary directories.
- The core Elasticsearch and Zipkin data sources and core Prometheus Azure and
  SigV4 authentication are removed in later releases. Package or replace what
  the deployment still needs.

## Feature-toggle hygiene

Remove obsolete toggles instead of retaining dead configuration. Grafana 13
supports direct per-toggle environment variables and deprecates the aggregate
`GF_FEATURE_TOGGLES_ENABLE` and `[feature_toggles] enable` forms. Provisioning,
Git Sync, dashboard restore, gzip, and several API-server behaviors have also
changed defaults; pin a setting explicitly when an external proxy, security
policy, or operational runbook depends on the former behavior.

## Validation after changes

Verify all of the following against the upgraded instance:

- health, startup migrations, database headroom, and rollback artifacts;
- dashboard and folder counts, UIDs, ownership, Git state, and home dashboard;
- alert evaluation, state recovery, notification delivery, templates, and HA;
- SSO, service accounts, SCIM, anonymous access, and custom roles;
- data-source connectivity from the server side, headers, TLS, and query modes;
- plugin signatures, compatibility, manifests, process configuration, and UI;
- logs, traces, profiles, transformations, exports, screenshots, and reports;
- metrics and dashboards that consume renamed or removed Grafana metrics.
