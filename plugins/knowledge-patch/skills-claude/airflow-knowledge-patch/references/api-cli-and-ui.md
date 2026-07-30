# API, CLI, and UI

## REST API v2 migration

The 3.0-upgrade replaces REST `/api/v1` with the FastAPI stable `/api/v2`.
Clients must treat validation failures as HTTP 422, send `logical_date` rather
than `execution_date`, and expect an omitted trigger date to remain `None`
(3.0.0).

Use `airflowctl`, distributed in `apache-airflow-client`, for remote
administration such as triggering Dags and managing Connections. Keep the
`airflow` CLI for local operations. Tokens are issued at `/auth/token`, and
auth-manager endpoints live under `/auth`.

## Waiting for runs and returning results

Since 3.1.0,
`GET /api/v2/dags/{dag_id}/dagRuns/{dag_run_id}/wait` streams repeated JSON
updates as NDJSON until completion. Its `result` query parameter can select a
task's XCom:

```bash
curl -H "Accept: application/x-ndjson" \
  "http://localhost:8080/api/v2/dags/ml_pipeline/dagRuns/manual_2024_01_15/wait?result=inference_task"
```

Airflow 3.3.0 can instead return the Dag-designated `@result` or marked
return-value XCom.

## Search, bulk changes, and external management

The 3.2.0 API supports OR search terms and Dag filtering by timetable type.
Bulk task-instance endpoints accept wildcard `dag_id` and `dag_run_id`;
task-instance search accepts `operator_name_pattern`, `pool_pattern`, and
`queue_pattern`; bulk PATCH accepts `update_mask`. A TaskInstance API supports
systems that manage tasks externally.

`api.page_size` is deprecated in favor of `api.fallback_page_limit` as of
3.2.0.

Airflow 3.3.0 adds `clearPartitions` and bulk
`/dags/{dag_id}/clearDagRuns`, including `partition_key` and `partition_date`
window selectors.

## Backfill authorization and operations

Backfills are scheduler-managed and observable through REST and UI from 3.0.0.
In 3.2.0, `BaseAuthManager.is_authorized_backfill` is removed. Authorize
backfills through `requires_access_dag` for `DagAccessEntity.Run`; policies
that granted Backfill permission but not Dag-run permission must change.

## Asset and UI endpoint changes

Asset responses use `scheduled_dags` instead of `consuming_dags` in 3.1.0.
The value represents Dags scheduling on the Asset, not every Dag that uses it.

Task-instance summaries use one NDJSON stream in 3.2.0:

```text
GET /ui/grid/ti_summaries/{dag_id}?run_ids=...
```

It returns one `GridTISummaries` JSON line per run. The former single-run
`/ui/grid/ti_summaries/{dag_id}/{run_id}` endpoint is removed.

The 3.2.0 UI can add, edit, and delete XCom values, and HITL details show the
full approval and rejection history.

## HITL and UI extensions

HITL tasks added in 3.1.0 wait for an authorized UI or API response. Their
forms may show Dag parameters and XCom values, and notification utilities can
link responders to the required-action page. In 3.3.0, `awaiting_input`
identifies this state explicitly.

React Apps and their dashboard/menu integrations are an experimental plugin
surface in 3.1.0. Backend plugins also gain `iframe_views` for navigation and
Dag-page external views.

In 3.3.0, plugin navigation can request `nav_top_level`, while `/auth` and
`/pluginsv2` are reserved prefixes. Owner-link and extra-link `href` values
must be HTTP, HTTPS, `mailto`, or relative URLs.

## CLI migrations and safer output

Removed spellings in 3.0.0 include:

- Replace `--ignore-depends-on-past` with `--depends-on-past ignore`.
- Pass `dag_id` positionally to `airflow dags list-runs`.
- Replace `airflow tasks list --tree` with `airflow dag show`.

Since 3.2.0, `connections list` and `variables list` hide sensitive values by
default and accept `--show-values` and `--hide-sensitive`. The removed
`connections list --conn-id` is replaced by `airflow connections get`.

Also in 3.2.0, Dag clear accepts `only_new`; `pools import` and
`connections import` accept `--action-on-existing-key`; `airflow db init`
again accepts `--use-migration-files`; and database cleaning can explicitly
include or exclude Dags. Partition ranges reach clear and backfill commands in
3.3.0.

Development servers support `--dev` hot reload in 3.2.0. `auth list-envs`
reports configured CLI environments and authentication status, while the UI
can generate JWTs for API and CLI use.

## Transport and connection security

API clients and servers support mutual TLS and private certificate authorities
in 3.3.0. CORS credential behavior is configurable, and wildcard origins are
rejected when credentials would make them unsafe. Connection tests may run
asynchronously on workers so the API server does not need direct Connection
access.
