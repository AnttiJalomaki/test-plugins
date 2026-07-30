# Migration and configuration

## Public compatibility boundary

During the `3.0-upgrade`, move Dag-authoring and task-runtime imports to the
semver-governed `airflow.sdk` surface:

```python
from airflow.sdk import Asset, DAG, dag, get_current_context, task
```

- Rename `Dataset*` imports to `Asset*`.
- Move `airflow.io.*` imports to `airflow.sdk.io.*`.
- Treat APIs not listed as public, the metadata schema, and Web UI HTML as
  internal implementation details.
- Base extension interfaces are public. For built-in operators, parameters
  and documented behavior are stable, but methods and structure are not.
- Built-in executor implementations are not supported subclassing contracts.

Task-facing exceptions moved to `airflow.sdk.exceptions` in 3.2.0; the old
`airflow.exceptions` proxies warn. Providers that bridge versions can use
`airflow.providers.common.compat.sdk`.

```python
from airflow.sdk.exceptions import AirflowSkipException, TaskDeferred
```

Invalid sensor `poke_interval` or `timeout` arguments now raise `ValueError`
rather than `AirflowException`.

The legacy `airflow.datasets`, `airflow.timetables.datasets`, and
`airflow.utils.dag_parsing_context` modules are removed in 3.2.0. Use their
Airflow 3 SDK-era replacements.

## Task access to deployment data

Tasks and workers communicate through the Task Execution API and cannot use
metadata ORM models or sessions. Use context/SDK accessors at runtime:

```python
from airflow.sdk import get_current_context

context = get_current_context()
ti = context["ti"]
connection = context["conn"].get("service")
variable = context["var"].value.get("setting")
```

For wider access to Dag runs, task instances, Connections, Variables, or
XComs, use stable REST endpoints or `apache-airflow-client`. Obtain a client
token from `/auth/token`.

## Upgrade preflight

Upgrade first to Airflow 2.7 or later, preferably the newest available 2.x.
Back up the metadata database, optionally clean it, and ensure Dag parsing and
reserialization complete without errors:

```bash
airflow db clean
airflow dags reserialize
```

Ruff 0.13.1 or later supplies Airflow migration rules. AIR301/AIR302 identify
breaks; AIR311/AIR312 identify recommended migrations. Import-path rewrites
may require unsafe fixes, followed by F401 cleanup of obsolete imports:

```bash
ruff check dags/ --select AIR301 --show-fixes
ruff check dags/ --select AIR301 --fix --unsafe-fixes
```

Install `apache-airflow-providers-standard` for common operators, sensors, and
triggers moved out of core, including `BashOperator`, `PythonOperator`,
`ExternalTaskSensor`, and `FileSensor`. The provider works on Airflow 2.x, so
move imports before upgrading core.

Diagnose and optionally rewrite configuration, then migrate the database:

```bash
airflow config update --fix
airflow db migrate
airflow config lint
```

The API server replaces the webserver process. The Dag processor must run as a
separate process even for local development:

```bash
airflow api-server
airflow dag-processor
```

## Removed architecture

- Replace SubDAGs with TaskGroups, Assets, or data-aware scheduling.
- Replace SequentialExecutor with LocalExecutor, which supports SQLite.
- Replace CeleryKubernetes/LocalKubernetes hybrids with multiple-executor
  configuration.
- Replace SLAs with Deadline Alerts.
- Replace CLI `--subdir`/`-S` selection with Dag bundles.
- Move REST clients from `/api/v1` to the FastAPI stable `/api/v2`.

## Authentication and plugin migration

Simple Auth is the default auth manager. To retain FAB, install its provider
and configure:

```ini
[core]
auth_manager = airflow.providers.fab.auth_manager.fab_auth_manager.FabAuthManager
```

Custom security managers import `FabAirflowSecurityManagerOverride` from
`airflow.providers.fab.auth_manager.security_manager.override`.

Auth-manager routes are under `/auth`. Update external OAuth callbacks such as
`/oauth-authorized/google` to `/auth/oauth-authorized/google`.

Plugins using `appbuilder_views`, `appbuilder_menu_items`, or
`flask_blueprints` must install the FAB provider compatibility layer or move to
`external_views`, `fastapi_apps`, and `fastapi_root_middlewares`. For Helm
deployments, move values under `webserver` to `apiServer` and audit every
renamed or removed option during the chart upgrade.

## API server and parser configuration

In 3.0.0, API server settings moved from `[webserver]` to `[api]`:

| Old key | New key |
| --- | --- |
| `web_server_host` | `host` |
| `web_server_port` | `port` |
| `web_server_worker_timeout` | `worker_timeout` |
| `web_server_ssl_cert` | `ssl_cert` |
| `web_server_ssl_key` | `ssl_key` |

`workers` and `access_logfile` retained their names at that point. Dag parsing
keys moved to `[dag_processor]`: `dag_file_processor_timeout`,
`parsing_processes`, `file_parsing_sort_mode`, `max_callbacks_per_loop`,
`min_file_process_interval`, `stale_dag_threshold`, and
`print_stats_interval`. Other legacy `[webserver]` settings and obsolete
scheduler/logging keys have no effect; locate them with `airflow config lint`.

In 3.1.0, move these additional settings from `[webserver]` to `[api]`:

- `log_fetch_timeout_sec`
- `hide_paused_dags_by_default`
- `page_size`
- `default_wrap`
- `require_confirmation_dag_change`
- `auto_refresh_interval`

`[api] access_logfile` was replaced by `[api] log_config`, which names a
`logging.config.fileConfig`-compatible file. `[api] workers` defaults to `1`;
prefer multiple API-server instances for horizontal scaling. Remove unused
`instance_name_has_markup`, `warn_deployment_exposure`, and
`dag_stale_not_seen_duration` settings.

In 3.2.0, `api.page_size` is deprecated in favor of
`api.fallback_page_limit`. Rendered-field retention is Dag-run based:
`max_num_rendered_ti_fields_per_task` is renamed to
`num_dag_runs_to_retain_rendered_fields`. It retains records for the newest
Dag runs, so sparse or conditional tasks can retain fewer rendered instances.

Individual secrets-backend arguments can be set independently as
`AIRFLOW__SECRETS__BACKEND_KWARG__<KEY>` instead of packing all arguments into
one combined configuration value.

## Scheduling defaults that affect migration

`catchup_by_default` now defaults to `False`. `create_cron_data_intervals`
also defaults to `False`, so a bare cron `schedule=` uses
`CronTriggerTimetable` rather than `CronDataIntervalTimetable`. Set it to
`True` before upgrading when tasks depend on interval boundaries or derived
`ds`/`ts` values. If it is re-enabled after Airflow 3 runs already exist, one
scheduled run is skipped to avoid a duplicate `logical_date`.

Removed context keys include `tomorrow_ds`, `tomorrow_ds_nodash`,
`yesterday_ds`, `yesterday_ds_nodash`, `prev_ds`, `prev_ds_nodash`,
`prev_execution_date`, `prev_execution_date_success`, `next_execution_date`,
`next_ds`, `next_ds_nodash`, and `execution_date`.

For manual runs, a supplied `logical_date` is not necessarily equal to the
timetable-resolved interval. Use it for the requested trigger date; reserve
`data_interval_start` and `data_interval_end` for actual interval semantics.

```python
requested_date = get_current_context()["logical_date"]
```

## Runtime, database, and image compatibility

In 3.1.0, Python 3.9 support is removed; supported runtimes are Python
3.10–3.13. SQLAlchemy 2.0 and the psycopg3 PostgreSQL driver are supported.

In 3.2.0, Python 3.14 is supported and SQLAlchemy 2 is the only supported
major line. Official container images no longer include a MySQL client; add it
to a derived image when operational tooling needs it.
