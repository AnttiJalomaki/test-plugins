---
name: airflow-knowledge-patch
description: Apache Airflow
version: 3.3.0
license: MIT
metadata:
  author: Nevaberry
---


# Apache Airflow Knowledge Patch

Use this skill when upgrading, authoring, integrating, or operating Apache
Airflow 3 deployments. Inspect the installed core and provider versions before
applying version-dependent advice. Prefer public `airflow.sdk` interfaces,
stable REST APIs, and observed project behavior over internal implementation
details.

## Reference index

| Reference | Topics |
| --- | --- |
| [upgrade-and-compatibility.md](references/upgrade-and-compatibility.md) | Airflow 3 migration, public API boundaries, imports, configuration, runtimes, serialization, and removed facilities |
| [task-authoring-and-execution.md](references/task-authoring-and-execution.md) | Task context, XCom, callbacks, retries, templates, state stores, async and non-Python execution |
| [scheduling-assets-and-deadlines.md](references/scheduling-assets-and-deadlines.md) | Scheduling defaults, backfills, partitions, assets, deadlines, multi-team behavior, and Dag versions |
| [api-cli-and-ui.md](references/api-cli-and-ui.md) | REST API v2, `airflowctl`, CLI migrations, UI endpoints, bulk operations, and transport security |
| [operations-logging-and-extensions.md](references/operations-logging-and-extensions.md) | API server, Dag processor, logging, metrics, bundles, plugins, triggerers, containers, and deployment controls |

## Migration-critical changes

### Author against the stable SDK

Move Dag-authoring and task-runtime imports to `airflow.sdk`. Rename Dataset
types to Asset types and move `airflow.io.*` imports to `airflow.sdk.io.*`.

```python
from airflow.sdk import Asset, DAG, dag, get_current_context, task
```

Only listed public interfaces are semver-governed. Metadata models, database
sessions, Web UI HTML, built-in executor implementations, and the methods or
structure of built-in operators are not extension contracts.

### Remove task-side metadata database access

Tasks communicate through the Task Execution API. Replace ORM and session use
with Task Context or SDK accessors. Use REST API v2 or
`apache-airflow-client` for broader access; obtain client tokens from
`/auth/token`.

```python
context = get_current_context()
connection = context["conn"].get("service")
variable = context["var"].value.get("setting")
```

### Update imports and dependencies

Install `apache-airflow-providers-standard` for common operators and sensors,
including `BashOperator`, `PythonOperator`, `ExternalTaskSensor`, and
`FileSensor`. Task exceptions now come from `airflow.sdk.exceptions`, and
serialization from `airflow.sdk.serde` and `airflow.sdk.serde.serializers`.

```python
from airflow.sdk.exceptions import AirflowSkipException, TaskDeferred
```

Plugins cannot register operators, sensors, hooks, or executors. Package and
import those classes directly. Do not subclass built-in executors as a stable
extension mechanism.

### Run the Airflow 3 preflight

Upgrade through Airflow 2.7 or later, back up the metadata database, confirm
clean parsing and reserialization, and run the migration checks.

```bash
airflow db clean
airflow dags reserialize
ruff check dags/ --select AIR301 --show-fixes
ruff check dags/ --select AIR301 --fix --unsafe-fixes
airflow config update --fix
airflow db migrate
```

Ruff 0.13.1 or later supplies AIR301/AIR302 breaking-change checks and
AIR311/AIR312 recommended migrations. Import rewrites can require unsafe fixes
and F401 cleanup.

### Start the new services

Replace `airflow webserver` with `airflow api-server`. Run the Dag processor as
a separate process, including in local development.

```bash
airflow api-server
airflow dag-processor
```

Use `airflow` for local administration and `airflowctl` from
`apache-airflow-client` for remote operations. Migrate REST clients from
`/api/v1` to `/api/v2`.

### Replace removed concepts

- Replace SubDAGs with TaskGroups, Assets, or data-aware scheduling.
- Replace SequentialExecutor with LocalExecutor, including on SQLite.
- Replace CeleryKubernetes/LocalKubernetes hybrids with multiple executors.
- Replace SLAs with Deadline Alerts and `fail_stop` with `fail_fast`.
- Replace CLI `--subdir`/`-S` selection with Dag bundles.
- Remove Dag and XCom pickling assumptions; Dags are JSON-serialized and
  non-native XCom representations require a custom backend.
- Stop importing `airflow.datasets`, `airflow.timetables.datasets`, or
  `airflow.utils.dag_parsing_context`.

## Scheduling and context changes

`catchup_by_default` and `create_cron_data_intervals` default to `False`. A
bare cron schedule therefore uses `CronTriggerTimetable`; explicitly enable
cron data intervals when code depends on interval-derived dates. Re-enabling
them after Airflow 3 runs exist skips one scheduled run to avoid duplication.

Many legacy date context keys, including `execution_date`, `prev_ds`, and
`next_ds`, are removed. Manual-run `logical_date` is not necessarily the
timetable-resolved data interval. Asset- or REST-triggered runs can have no
logical date or data interval, so guard `dag_run.logical_date` and absent
context keys.

`ti.xcom_pull(key=...)` searches the current task by default. Always pass
`task_ids` when pulling another task's value.

```python
value = ti.xcom_pull(task_ids="upstream_task", key="shared_state")
```

## High-value authoring features

### Partition-aware assets

Partitioned Assets support validated and chained key mappings, temporal
windows, fan-out, rollup, categorical fixed keys, runtime-assigned keys, and
wait policies. Producer partition dates and emitted keys propagate to
consumers. Apply global or per-mapper downstream-key limits to bound fan-out.

Use typed `Asset`, `AssetAlias`, or `Asset.ref` keys with event maps; string
keys are invalid. Shared aliases can be created across Dag files.

### Durable task and asset state

Use `task_state_store` and `asset_state_store` for JSON state with `get`,
`set`, `delete`, and `clear`. Configure expiration, retention, row-size limits,
garbage collection, `clear_on_success`, and an optional worker-side backend.
Task state survives retries and runs; asset state is available to triggers.

### Flexible task execution

- `PythonOperator` accepts `async def` callables.
- Async tasks have native async XCom access and `BaseHook.aget_hook()`.
- `@task.stub` declares implementations outside Python.
- Experimental coordinators run Java or native executables such as Go while
  using the Execution API for Variables, Connections, and XComs.
- Custom retry policies can decide whether and when to retry by exception;
  numeric `retry_exponential_backoff` values select the multiplier.
- `ResumableJobMixin` lets supported external jobs, initially Spark submit,
  resume after worker loss; `durable` opts out.

### Human decisions and deadlines

HITL operators defer for authorized UI or API responses and can expose Dag
parameters or XCom-backed form context. Waiting tasks use the
`awaiting_input` state, and `airflow dags test` waits for input.

Deadline Alerts support asynchronous and synchronous callbacks, callback
lists, executor selection, names, Variable-resolved intervals, Core API
endpoints, and callback timeouts. Check the installed version before using
synchronous callbacks or runtime Connections and Variables.

### Dag results and run watching

Use `@result` or mark a return-value XCom as the designated Dag result. The
NDJSON Dag-run wait endpoint can stream status and return either a named task
XCom or the designated result without client-side polling.

## Operational guardrails

- Audit `[webserver]` settings: API server options move to `[api]` and parser
  options to `[dag_processor]`. Use `airflow config lint`.
- Sensitive Connection and Variable values are hidden in CLI listings unless
  explicitly shown.
- Prefer multiple API-server instances for scale. Gunicorn is optional when
  preloading and rolling worker recycling are required.
- Remote logging now resolves through custom logging configuration, provider
  `RemoteLogIO`, then a transitional local-settings fallback.
- OpenTelemetry timers are Histograms; account for the newer Dag-processing
  tags and explicitly choose whether to retain legacy metric names.
- Multi-team deployments remain experimental; enforce team scope consistently
  across Dags, secrets, assets, pools, triggers, XCom, and asset queries.
- Configure mutual TLS, private certificate authorities, and CORS credentials
  deliberately. Credentialed CORS must not use wildcard origins.

## Working method

1. Read the project manifests and constraints to determine core and provider
   versions.
2. Choose the relevant topic reference from the index before editing code or
   configuration.
3. Separate public SDK/API usage from internal server implementation details.
4. Apply only features available in the installed version, especially for
   experimental partitions, teams, HITL, coordinators, and deadlines.
5. Validate Dag parsing, serialization, configuration lint, database migration,
   REST schemas, and operational metrics in the project's own environment.
