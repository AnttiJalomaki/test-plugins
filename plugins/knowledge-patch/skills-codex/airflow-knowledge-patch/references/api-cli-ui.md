# APIs, CLI, and UI

## REST API v2 request behavior

Use the stable FastAPI-based `/api/v2`; `/api/v1` is removed. Since 3.0.0,
clients must:

- handle validation failures as HTTP 422;
- send `logical_date` rather than `execution_date`;
- expect an omitted trigger date to remain `None`.

An event-driven run may have no logical date or data interval. Do not infer a
date that the server omitted.

Remote operations such as triggering Dags and managing Connections moved to
`airflowctl`, distributed with `apache-airflow-client`. The `airflow` CLI is
for local deployment operations.

## Scheduler-managed backfills

Backfills are no longer separate CLI jobs in 3.0.0. The scheduler manages them
as ordinary Dag runs, including structure versioning and observability. Start
and monitor backfills through the UI or REST API.

Backfill authorization changed in 3.2.0. `BaseAuthManager.is_authorized_backfill`
is removed; access flows through `requires_access_dag` for
`DagAccessEntity.Run`. Update policies that granted Backfill permission but not
Dag-run permission.

## Watch a Dag run as NDJSON

Since 3.1.0, this endpoint repeatedly emits JSON updates until completion:

```text
GET /api/v2/dags/{dag_id}/dagRuns/{dag_run_id}/wait
```

Set `Accept: application/x-ndjson`. The `result` query parameter can include an
XCom result, which supports quasi-synchronous integrations without custom
status polling:

```bash
curl -H "Accept: application/x-ndjson" \
  "http://localhost:8080/api/v2/dags/ml_pipeline/dagRuns/manual_2024_01_15/wait?result=inference_task"
```

In 3.3.0, a Dag can designate its own result with `@result` or a marked
return-value XCom; the wait API can return that result instead of an
arbitrarily named task XCom.

## Search, bulk updates, and external task control

REST capabilities added in 3.2.0 include:

- OR expressions in search parameters;
- Dag filtering by timetable type;
- wildcard `dag_id` and `dag_run_id` in bulk task-instance endpoints;
- task-instance filters `operator_name_pattern`, `pool_pattern`, and
  `queue_pattern`;
- `update_mask` on bulk PATCH endpoints.

An exposed `TaskInstance` API supports systems that manage tasks externally.

Airflow 3.3.0 adds partition-aware operations. `clearPartitions` and bulk
`/dags/{dag_id}/clearDagRuns` accept `partition_key` and `partition_date`
window selectors.

## Pagination

`api.page_size` is deprecated as of 3.2.0. Configure
`api.fallback_page_limit` for requests that do not provide their own limit.

## UI stream and state changes

Task-instance summaries use one NDJSON stream in 3.2.0:

```text
GET /ui/grid/ti_summaries/{dag_id}?run_ids=...
```

It emits one `GridTISummaries` JSON line per run. The former single-run
`/ui/grid/ti_summaries/{dag_id}/{run_id}` endpoint is removed.

The UI can add, edit, and delete XCom values. HITL task details show complete
approval and rejection history. In 3.3.0, HITL tasks waiting on a responder
use `awaiting_input`, which gives monitoring and automation a distinct state.

The UI can generate JWTs for API and CLI access as of 3.2.0.

## CLI spelling migrations

Deprecated spellings removed in 3.0.0:

| Removed form | Replacement |
| --- | --- |
| `--ignore-depends-on-past` | `--depends-on-past ignore` |
| `airflow dags list-runs --dag-id ID` | Pass `dag_id` positionally |
| `airflow tasks list --tree` | `airflow dag show` |

CLI behavior added or changed in 3.2.0:

- `connections list` and `variables list` hide sensitive values by default.
  Use `--show-values` or `--hide-sensitive` explicitly.
- `connections list --conn-id` is removed; use `airflow connections get` for
  one Connection.
- Dag clear accepts `only_new` to clear only newly added task instances.
- `pools import` and `connections import` accept
  `--action-on-existing-key`.
- `airflow db init` again accepts `--use-migration-files`.
- Database cleaning can explicitly include or exclude Dags.
- `airflow scheduler --num-runs 1 --only-idle` counts a run only while idle,
  allowing queued work and triggered Dags to finish before exit.
- The `trigger` command accepts `--queues` to route triggers by task queue to
  selected Triggerer hosts.
- Local development supports hot reload with `--dev`.
- `airflow auth list-envs` reports configured CLI environments and their
  authentication status.

Partition-aware clearing and backfills gain CLI partition ranges in 3.3.0.
Pool commands accept `--team-name` in multi-team deployments.

## API transport security

As of 3.3.0, API clients and servers support mutual TLS and private certificate
authorities. CORS credential handling is configurable, and wildcard origins
are rejected when credentials would make them unsafe.

Connection tests can execute asynchronously on workers, isolating access to
Connections from the API server.
