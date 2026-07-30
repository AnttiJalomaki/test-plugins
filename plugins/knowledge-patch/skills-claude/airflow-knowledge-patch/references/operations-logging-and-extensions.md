# Operations, Logging, and Extensions

## Executors, processes, and workers

The Edge Executor is generally available in 3.0.0. It runs tasks in remote or
distributed environments through the Task Execution API, supporting hybrid
deployments with workers near their data or applications.

Airflow 3 replaces the webserver process with `airflow api-server` and requires
a separately running Dag processor. Uvicorn is the default API server. In
3.2.0, installing `apache-airflow-core[gunicorn]` and selecting Gunicorn adds
preloaded memory-sharing workers and zero-downtime FIFO recycling:

```ini
[api]
server_type = gunicorn
worker_refresh_interval = 43200
worker_refresh_batch_size = 1
```

Uvicorn does not provide rolling restarts. Prefer multiple API-server instances
for horizontal scaling.

Airflow 3.3.0 makes multiprocessing start methods configurable globally with
`[core] mp_start_method` and `[core] mp_forkserver_preload`, with per-component
overrides in `[scheduler]`, `[triggerer]`, and `[dag_processor]`.

## Logging

Task, hook, and operator `LoggingMixin.log` is a structlog logger in 3.1.0.
Standard-library calls remain valid, while structlog supports searchable
fields:

```python
self.log.info("Registering adapter", name=item.name)
```

Set `[logging] json_logs = True` in 3.2.0 for newline-delimited JSON API-server
access logs, warnings, exceptions, and other output. `airflow celery worker`
does not yet support this mode.

Other 3.2.0 controls include `log_timestamp_format` for component timestamps
and `uvicorn_logging_level` for API access-log verbosity. The Execution API
propagates correlation IDs.

## Remote logging

In 3.3.0, remote-log handlers resolve in this order:

1. Custom `[logging] logging_config_class`.
2. A provider `RemoteLogIO` selected from the `remote_base_log_folder` scheme.
3. The transitional `airflow_local_settings.py` fallback.

Provider handlers need a no-argument `from_config()` method. Resolution is
lazy, callback subprocesses can upload remote logs, and
`airflow.logging_config.load_logging_config()` is deprecated.

## Metrics and tracing

The `executor.running_dags` gauge added in 3.2.0 reports running Dags.

OpenTelemetry timers become Histograms in 3.3.0.
`dag_processing.last_run.seconds_ago` adds `file_path`, `bundle_name`, and
`file_name` tags. Filename-suffixed legacy metrics remain enabled unless
`[metrics] legacy_names_on` is disabled. Head sampling is supported, and
custom samplers receive `dag_id`, `run_id`, and `run_type` attributes.

## Dag bundles and parsing tools

Dag structures are versioned in 3.0.0. Because the triggerer does not
initialize Dag bundles, bundle-only trigger classes are invalid; put trigger
implementations somewhere else on `sys.path`.

`FileLoadStat` gains nullable `bundle_path` and `bundle_name` in 3.2.0. Paths
are genuine relative paths and no longer use a leading `/` to mean “relative
to the Dags directory”; custom tooling should use `pathlib.Path`.

`GitDagBundle` supports submodules and HTTP URL authentication in 3.2.0.

In 3.3.0, each provider's example Dags live in a separate bundle named
`apache-airflow-providers-<distribution>-example-dags`, or
`<distribution>-example-dags` for third-party providers. API filters that
assumed examples lived in `dags-folder` must change. `[core] load_examples`
still controls registration.

## Triggerers and lifecycle hooks

Airflow 3.2.0 adds trigger-command `--queues` to route triggers by task queue
and `max_trigger_to_select_per_loop` to bound HA Triggerer selection.

In 3.3.0, `BaseTrigger.on_kill()` handles user actions against a trigger.
`task_instance_mutation_hook` now receives the associated `DagRun`.
Triggerers can also be assigned and filtered by team.

## Secrets and deployment controls

Individual secrets-backend arguments can be set in 3.2.0 with
`AIRFLOW__SECRETS__BACKEND_KWARG__<KEY>` instead of supplying one combined
kwargs value.

Official 3.2.0 images omit the MySQL client. Add it to derived images when
needed. Container builds expose `PYTHON_LTO` for FIPS compatibility and can
verify cryptographic signatures on Python source packages.

The API client/server support mutual TLS, private CAs, and configurable CORS
credentials in 3.3.0. Credentialed CORS rejects wildcard origins. Asynchronous
worker-side Connection tests can isolate Connection access from the API server.

## Serialization and rolling upgrades

Airflow 3.1.0 adds versioned Dag-serialization contracts so separately
deployed components can upgrade with less coordination. This is a foundation
for decoupling, not complete separation; the latter was planned for 3.2.

Custom serializer deserializers receive an already loaded class rather than a
class-name string. In 3.2.0, migrate serde imports to the Task SDK paths;
legacy paths warn and are scheduled for removal in Airflow 4.

## Retention and maintenance

`max_num_rendered_ti_fields_per_task` is deprecated and renamed to
`num_dag_runs_to_retain_rendered_fields` in 3.2.0. Retention counts newest Dag
runs, not task executions, so sparse or conditional tasks may retain fewer
rendered records.

Database cleaning can include or exclude selected Dags. State-store retention
in 3.3.0 has configurable expiration, garbage collection, and row-size limits.
