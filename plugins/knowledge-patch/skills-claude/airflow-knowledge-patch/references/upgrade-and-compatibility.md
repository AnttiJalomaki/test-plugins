# Upgrade and Compatibility

## Public API boundary and preflight

Airflow 3 establishes `airflow.sdk` as the semver-governed public surface for
Dag authoring and task execution (3.0-upgrade). Move decorators and core types
there, rename `Dataset*` imports to `Asset*`, and move `airflow.io.*` to
`airflow.sdk.io.*`.

Unlisted Python APIs, the metadata schema, and Web UI HTML are internal. Base
extension interfaces are public, but only a built-in operator's parameters and
behavior—not its methods or structure—are stable. Built-in executor
implementations are not safe subclassing contracts.

Tasks and workers use the Task Execution API and can no longer open metadata
ORM sessions. Use Task Context or SDK accessors at runtime. For Dag runs, task
instances, Connections, Variables, and XComs beyond task context, use REST API
v2 or `apache-airflow-client`; obtain a token from `/auth/token`.

Before upgrading, reach at least Airflow 2.7, preferably the latest 2.x, back
up and optionally clean the metadata DB, and ensure Dag parsing and
reserialization succeed:

```bash
airflow db clean
airflow dags reserialize
ruff check dags/ --select AIR301 --show-fixes
ruff check dags/ --select AIR301 --fix --unsafe-fixes
```

Ruff 0.13.1 or later provides AIR301/AIR302 for breaks and AIR311/AIR312 for
recommended migrations. Import changes can require `--unsafe-fixes`; use F401
to remove stale imports.

## Dependencies and import moves

Common operators, sensors, and triggers formerly shipped in core require
`apache-airflow-providers-standard`, including `BashOperator`,
`PythonOperator`, `ExternalTaskSensor`, and `FileSensor`. Install the provider
on Airflow 2.x to migrate imports before upgrading core.

Task-facing exceptions moved to `airflow.sdk.exceptions` in 3.2.0. Imports
from `airflow.exceptions` warn, and providers may use
`airflow.providers.common.compat.sdk`. Invalid sensor `poke_interval` or
`timeout` values now raise `ValueError`, not `AirflowException`.

Serialization moved to `airflow.sdk.serde` and
`airflow.sdk.serde.serializers.*` in 3.2.0. The old
`airflow.serialization.*` paths warn and remain only until Airflow 4. Custom
deserializers have received the loaded class rather than a class-name string
since 3.1.0:

```python
def deserialize(cls: type, version: int, data: Any):
    ...
```

`get_task_group_children_getter` and `task_group_to_dict` are no longer public
and moved from `airflow.sdk.definitions.taskgroup` into server-side services
in 3.1.0. `PriorityWeightStrategy.serialize()` and `.deserialize()`, plus
internal `TaskInstance.run()`, `.render_templates()`,
`.get_template_context()`, and related private members, were removed in 3.2.0.

The legacy `airflow.datasets`, `airflow.timetables.datasets`, and
`airflow.utils.dag_parsing_context` modules are removed in 3.2.0. Use their
SDK-era Asset and parsing surfaces.

## Configuration and service migration

Use `airflow config update` to diagnose changes and optionally apply them with
`--fix`, then run `airflow db migrate`. Replace the webserver process with
`airflow api-server`, and run `airflow dag-processor` separately even in local
development.

The initial 3.0.0 moves from `[webserver]` to `[api]` are:

| Old key | New key |
| --- | --- |
| `web_server_host` | `host` |
| `web_server_port` | `port` |
| `web_server_worker_timeout` | `worker_timeout` |
| `web_server_ssl_cert` | `ssl_cert` |
| `web_server_ssl_key` | `ssl_key` |

`workers` and `access_logfile` initially retain their names. Move
`dag_file_processor_timeout`, `parsing_processes`, `file_parsing_sort_mode`,
`max_callbacks_per_loop`, `min_file_process_interval`, `stale_dag_threshold`,
and `print_stats_interval` into `[dag_processor]`. Obsolete scheduler/logging
keys and other legacy `[webserver]` options have no effect; find them with
`airflow config lint`.

In 3.1.0, also move `log_fetch_timeout_sec`,
`hide_paused_dags_by_default`, `page_size`, `default_wrap`,
`require_confirmation_dag_change`, and `auto_refresh_interval` from
`[webserver]` to `[api]`. Replace `[api] access_logfile` with `[api] log_config`
pointing to a `logging.config.fileConfig`-compatible file. `[api] workers`
defaults to `1`; prefer multiple API-server instances for horizontal scaling.
`instance_name_has_markup`, `warn_deployment_exposure`, and
`dag_stale_not_seen_duration` are removed.

In Helm values, move configuration under `webserver` to `apiServer` and review
all renamed or removed Airflow options.

## Authentication and extension migration

Simple Auth is the default auth manager in Airflow 3. To retain FAB, install
its provider and configure:

```ini
[core]
auth_manager = airflow.providers.fab.auth_manager.fab_auth_manager.FabAuthManager
```

Custom security managers import `FabAirflowSecurityManagerOverride` from
`airflow.providers.fab.auth_manager.security_manager.override`. Auth-manager
routes live under `/auth`; for example, `/oauth-authorized/google` becomes
`/auth/oauth-authorized/google`.

Plugins using `appbuilder_views`, `appbuilder_menu_items`, or
`flask_blueprints` must install the FAB compatibility provider or migrate to
`external_views`, `fastapi_apps`, and `fastapi_root_middlewares`. Operators,
sensors, hooks, and executors are ordinary Python classes and can no longer be
registered or imported via the plugin namespace; import their packages
directly. Provider hooks `get_connection_form_widgets` and
`get_ui_field_behaviour` are deprecated as of 3.2.0.

## Removed facilities

During the 3.0-upgrade:

- Replace SubDAGs with TaskGroups, Assets, or data-aware scheduling.
- Replace SequentialExecutor with LocalExecutor; LocalExecutor supports
  SQLite.
- Replace CeleryKubernetes/LocalKubernetes hybrids with multiple executors.
- Replace SLAs with Deadline Alerts.
- Replace CLI `--subdir` and `-S` with Dag bundles.
- Replace `/api/v1` with the FastAPI stable `/api/v2`.

Dag pickling and XCom pickling are removed in 3.0.0. Dags must be
JSON-serializable. Use a custom XCom backend for values needing another
representation.

## Runtime, database, and image compatibility

Airflow 3.1.0 removes Python 3.9 and supports Python 3.10–3.13. It adds
SQLAlchemy 2.0 compatibility and psycopg3 support. Airflow 3.2.0 adds Python
3.14 and supports only SQLAlchemy 2.

Official 3.2.0 container images no longer include a MySQL client. Add one to a
derived image when required. For FIPS-oriented builds, `PYTHON_LTO` controls
Python link-time optimization; builds can also verify cryptographic signatures
on Python source packages.
