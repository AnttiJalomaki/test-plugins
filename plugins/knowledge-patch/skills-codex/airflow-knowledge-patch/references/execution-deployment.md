# Execution and deployment

## Execution architecture

The Edge Executor is generally available in 3.0.0. It runs tasks in distributed
or remote compute environments through the Task Execution API, enabling hybrid
deployments with workers close to their data or applications.

Tasks cannot access metadata ORM models or sessions. Remote runtimes obtain
task-facing Variables, Connections, and XComs through the Execution API.

SequentialExecutor is removed; use LocalExecutor, including with SQLite.
CeleryKubernetesExecutor and LocalKubernetesExecutor hybrids are removed; use
multiple-executor configuration instead. Do not subclass built-in executor
implementations as if they were stable extension contracts.

## Scheduler-managed work

Backfills are scheduler-managed Dag runs, not standalone CLI jobs. Their Dag
versions and status are visible through normal UI and REST surfaces.

For run-limited scheduler jobs in 3.2.0, add `--only-idle` so `--num-runs`
counts only idle loops and the scheduler can finish processing triggered Dags
and queued tasks:

```bash
airflow scheduler --num-runs 1 --only-idle
```

## Triggerer routing

The 3.2.0 `trigger` command accepts `--queues`, routing triggers according to a
task queue to particular Triggerer hosts. Configure
`max_trigger_to_select_per_loop` to cap selection per loop in high-availability
Triggerer deployments.

The triggerer does not initialize Dag bundles. Custom triggers must be
importable independently on `sys.path`.

In 3.3.0, triggerers can also be assigned and filtered by team.
`BaseTrigger.on_kill()` handles user actions against a trigger.

## Task SDK runtime capabilities

Airflow 3.2.0 expands task runtime access:

- Create Connections from URIs through the SDK.
- Retrieve the previous task instance from `RuntimeTaskInstance`.
- Import `BaseXcom` from `airflow.sdk`.

Airflow 3.3.0 adds native async XCom accessors and `BaseHook.aget_hook()` for
async tasks. Worker-side Dag registrations let structured XCom outputs
round-trip as Pydantic model instances.

## Tasks implemented outside Python

`@task.stub` in 3.2.0 declares a task implemented in another language. In
3.3.0, the experimental Coordinator layer can dispatch such declarations:

- `JavaCoordinator` executes JVM tasks.
- `ExecutableCoordinator` executes native binaries such as Go programs.

The queue selects a coordinator. Non-Python runtimes use the Execution API;
Dag authoring and scheduling stay in Python.

## API server worker models

Uvicorn remains the default API server and cannot perform rolling restarts.
For preloaded, memory-sharing workers and zero-downtime FIFO recycling in
3.2.0, install `apache-airflow-core[gunicorn]` and configure:

```ini
[api]
server_type = gunicorn
worker_refresh_interval = 43200
worker_refresh_batch_size = 1
```

For horizontal scaling, prefer multiple API server instances; `[api] workers`
defaults to `1` in 3.1.0.

## Dag bundles

`GitDagBundle` supports repository submodules and HTTP URL authentication as
of 3.2.0.

Provider example Dags move to dedicated bundles in 3.3.0. Apache provider
bundles are named `apache-airflow-providers-<distribution>-example-dags`;
third-party bundles use `<distribution>-example-dags`. Clients that locate
examples by filtering for `dags-folder` must update their filters.
`[core] load_examples` still controls whether examples are registered.

Rerun bundle selection follows `rerun_with_latest_version`. Request/CLI input
wins over the Dag setting, then `[core]` configuration. If none is specified,
clear/rerun use the original version (`False`) and backfill uses the latest
version (`True`).

## Multi-team deployments

A single deployment can experimentally isolate each team's Dags,
Connections, Variables, pools, executors, resources, and permissions in
3.2.0.

Enforcement expands in 3.3.0:

- Asset access uses `access_control` rather than `allow_producer_teams`.
- `AssetAccessControl` includes `consumer_teams` and `allow_global`.
- XCom Execution API calls and asset queries are team-scoped.
- Pool scheduling enforces ownership; pool CLI commands accept `--team-name`.
- Triggerers can be assigned and filtered by team.

Treat the multi-team surface as experimental and test isolation in API,
scheduler, worker, pool, asset, XCom, and trigger paths.

## Multiprocessing

Airflow 3.3.0 adds `[core] mp_start_method` and
`[core] mp_forkserver_preload` for global multiprocessing behavior. Override
them per component in `[scheduler]`, `[triggerer]`, or `[dag_processor]`.

## Containers and official images

Airflow 3.2.0 official images no longer include a MySQL client; add it in a
derived image when required. Container build compliance controls include:

- `PYTHON_LTO` to make Python link-time optimization configurable for FIPS
  builds.
- Verification of cryptographic signatures on Python source packages.

## External job durability

`ResumableJobMixin` in 3.3.0 tracks external work across worker failure.
`SparkSubmitOperator` is the first integration. Resumption avoids starting the
external job twice; set `durable` to opt out where restart semantics are
preferred.

## External task management

The 3.2.0 `TaskInstance` API supports systems that manage tasks outside the
normal worker lifecycle. Correlation IDs propagated through the Execution API
can link those operations to component logs and traces.
