---
name: airflow-knowledge-patch
description: Apache Airflow
version: 3.3.0
license: MIT
metadata:
  author: Nevaberry
---


# Apache Airflow Knowledge Patch

Use this skill when authoring, upgrading, operating, or integrating Apache
Airflow and the work touches the Task SDK, API server, assets, scheduling,
executors, plugins, serialization, observability, or deployment behavior.

Prefer the project's installed Airflow and provider versions, configuration,
code, and tests when they disagree with this guidance. Treat experimental
surfaces as unstable and verify their exact signatures locally.

## Reference index

| Reference | Topics |
| --- | --- |
| [references/migration-configuration.md](references/migration-configuration.md) | Upgrade preflight, stable imports, removed facilities, auth, config moves, runtime compatibility |
| [references/dag-task-authoring.md](references/dag-task-authoring.md) | Dag and task semantics, callbacks, retries, deadlines, HITL, bundles, results |
| [references/assets-partitions.md](references/assets-partitions.md) | Assets, aliases, event maps, partition mapping, partition operations, state stores |
| [references/api-cli-ui.md](references/api-cli-ui.md) | REST and Execution APIs, `airflowctl`, CLI migrations, UI streams and controls |
| [references/execution-deployment.md](references/execution-deployment.md) | Executors, backfills, workers, triggerers, containers, bundles, multi-team deployments |
| [references/extensions-serialization.md](references/extensions-serialization.md) | Plugins, extension hooks, serialization, XCom, operator links, provider hooks |
| [references/observability-security.md](references/observability-security.md) | Logging, metrics, tracing, remote logs, API transport security |

## Breaking changes first

### Author through `airflow.sdk`

Use the semver-governed Task SDK for Dag authoring and task execution:

```python
from airflow.sdk import Asset, DAG, dag, get_current_context, task
```

Move `Dataset*` names to `Asset*`, `airflow.io.*` to `airflow.sdk.io.*`, task
exceptions to `airflow.sdk.exceptions`, and serialization to
`airflow.sdk.serde`. Do not depend on metadata schema, Web UI HTML, internal
operator methods, built-in executor implementation details, or other unlisted
Python APIs.

### Keep task code away from the metadata database

Workers use the Task Execution API. Task code must use Task Context and SDK
accessors for runtime data, or the stable REST API/client for broader
administration. It must not open metadata ORM sessions.

```python
from airflow.sdk import get_current_context

context = get_current_context()
ti = context["ti"]
connection = context["conn"].get("service")
variable = context["var"].value.get("setting")
```

Acquire client tokens from `/auth/token` when using `apache-airflow-client`.

### Replace removed architecture

- Run `airflow api-server`; the webserver command is gone.
- Run `airflow dag-processor` as a separate process, including locally.
- Use TaskGroups, Assets, or data-aware scheduling instead of SubDAGs.
- Use LocalExecutor instead of SequentialExecutor; SQLite is supported.
- Configure multiple executors instead of hybrid Kubernetes executors.
- Replace SLAs with Deadline Alerts and REST `/api/v1` with `/api/v2`.
- Replace CLI `--subdir`/`-S` selection with Dag bundles.
- Import operators, hooks, sensors, and executors from their packages rather
  than through the plugin namespace.

### Install the standard provider for moved components

Install `apache-airflow-providers-standard` for common operators and sensors,
including `BashOperator`, `PythonOperator`, `ExternalTaskSensor`, and
`FileSensor`. It can be installed before the core upgrade so imports can move
while still on the earlier deployment.

### Audit scheduling semantics

`catchup_by_default` is `False`. With `create_cron_data_intervals=False`, a
bare cron schedule uses `CronTriggerTimetable`, not
`CronDataIntervalTimetable`. Set the option explicitly when tasks rely on
interval boundaries or `ds`/`ts` derivations. Re-enabling interval creation
after runs exist skips one scheduled run to prevent a duplicate logical date.

Manual and event-driven runs require defensive date handling. A supplied
`logical_date` need not equal the timetable's data interval, and some runs
have neither a logical date nor interval fields. Inspect `dag_run.logical_date`
and guard missing context values.

### Make XCom reads explicit

An unqualified pull searches only the current task. Name the producer when
reading another task:

```python
value = ti.xcom_pull(task_ids="upstream_task", key="shared_state")
```

XCom pickling is unavailable. Use JSON-native values or a custom backend;
never rely on the API server deserializing unknown Python objects for display.

### Update renamed and removed task APIs

- Use Dag argument `fail_fast`; `fail_stop` is removed.
- Do not use `TriggerRule.ALWAYS` for teardown tasks. Teardowns still execute
  after early Dag termination but must preserve upstream dependency semantics.
- Do not expect `on_success_callback` for a task marked `SKIPPED`.
- Replace removed internal `TaskInstance.run()`, `.render_templates()`, and
  `.get_template_context()` usage with supported SDK/runtime surfaces.
- Use numeric `retry_exponential_backoff`; `0` disables it and booleans retain
  compatibility as `2.0`/`0.0`, although REST rejects booleans.

## Upgrade workflow

Start from Airflow 2.7 or later, preferably the newest 2.x. Back up the
metadata database, optionally clean it, then verify parsing and reserialization:

```bash
airflow db clean
airflow dags reserialize
ruff check dags/ --select AIR301 --show-fixes
ruff check dags/ --select AIR301 --fix --unsafe-fixes
```

Use Ruff 0.13.1 or later. AIR301/AIR302 locate breaks; AIR311/AIR312 recommend
migrations. Import moves may also need F401 cleanup. Then diagnose, apply, and
migrate configuration:

```bash
airflow config update --fix
airflow db migrate
airflow config lint
```

Review [references/migration-configuration.md](references/migration-configuration.md)
before changing Helm values, authentication, API server settings, parsing
settings, Python/SQLAlchemy versions, or official images.

## High-value capabilities

### Partition-aware assets

Partitioned asset scheduling can update only affected downstream partitions.
Compose and validate mappings with the supported mapper types; use fan-out,
rollup, categorical keys, runtime keys, and temporal windows deliberately.
Cap downstream key expansion globally and, where needed, per mapper. See
[references/assets-partitions.md](references/assets-partitions.md).

### Human-in-the-loop and deadlines

HITL operators defer for authorized UI or API input and expose response
history. A waiting task uses the distinct `awaiting_input` state. Deadline
alerts can use asynchronous callbacks or executor-backed synchronous
callbacks, callback lists, names, and resolved intervals; check the reference
for feature maturity and runtime-secret restrictions by release.

### Async and non-Python work

`PythonOperator` accepts `async def` callables. Async tasks have async XCom and
hook access. `@task.stub` declares externally implemented tasks; the
experimental Coordinator layer can route Java and native binaries while
authoring and scheduling remain in Python.

### Durable state and resumable external jobs

`task_state_store` and `asset_state_store` persist JSON state with expiration,
retention, deletion, and clearing controls. Task state can survive retries and
runs. For supported external jobs, `ResumableJobMixin` prevents duplicate work
after worker failure; `durable` opts an operator out.

### Streaming integrations and Dag results

The Dag-run wait endpoint emits NDJSON until completion and can include an
XCom result. Mark a TaskFlow task with `@result`, or designate its return-value
XCom, so clients can request the Dag-defined result rather than coupling to an
arbitrary task.

## Operating rules

1. Confirm installed core and provider versions before applying a feature.
2. Run `airflow config lint` after configuration changes.
3. Keep Dag/task code on public SDK interfaces and JSON-serializable values.
4. Test event-driven runs with missing logical-date and interval context.
5. Test pool, teardown, callback, and retry behavior under failure.
6. Validate API clients against 422 responses, nullable dates, pagination,
   authentication, and transport security.
7. Treat React Apps, multi-team deployment, Coordinator execution, and
   Deadline Alerts according to the maturity notes in the references.
8. For production changes, verify database migrations, worker/API rollout,
   remote logging, and rollback behavior in a staging deployment.
