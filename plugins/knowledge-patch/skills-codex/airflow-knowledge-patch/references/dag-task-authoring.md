# Dag and task authoring

## Dag structure and bundles

Dag structures are versioned in 3.0.0. Task renames, dependency changes, and
other structural edits are recorded in the metadata database, and historical
versions are visible through the UI and API.

The triggerer does not initialize Dag bundles. Trigger implementations cannot
come only from a bundle; place them in an importable package available on the
triggerer's `sys.path`.

For clearing, reruns, backfills, and `TriggerDagRunOperator` reruns, 3.3.0 adds
`rerun_with_latest_version`. The effective choice follows this precedence:

1. Request parameter or CLI flag.
2. Dag setting.
3. `[core] rerun_with_latest_version`.
4. Default: `False` for clear/rerun and `True` for backfill.

## Logical dates and data intervals

Future `logical_date` values are rejected. Pass `logical_date=None` to create
a run at the current time. Asset-triggered runs and REST-triggered runs that
omit a date keep it as `None`, have no data interval, and omit `logical_date`,
`data_interval_start`, and `data_interval_end` from task context. Inspect and
guard `dag_run.logical_date` rather than assuming these keys exist.

For a manual run, the resolved interval need not be derived from or equal to
the supplied logical date. Use `logical_date` when the Dag means the requested
trigger date and interval fields only when the timetable provides interval
semantics.

A Dag with `schedule="@continuous"` may omit `start_date` as of 3.2.0; it
starts immediately when unpaused.

## Failure, skips, pools, and teardown

- `on_success_callback` does not run for a task marked `SKIPPED`.
- A task's effective `priority_weight` is capped by available pool slots. A
  large weight cannot defeat the pool's resource constraint during ordering.
- Teardown tasks continue after early Dag-run termination.
- `TriggerRule.ALWAYS` is invalid on teardown tasks. Choose a rule that keeps
  upstream dependency semantics.
- Use Dag argument `fail_fast`; the former `fail_stop` name is removed.

The `ALL_DONE_MIN_ONE_SUCCESS` trigger rule, added in 3.1.0, runs after every
upstream task finishes when at least one succeeded. Skipped upstream tasks
keep normal skip propagation.

Dag callbacks receive a task instance relevant to the Dag's final state as of
3.2.0 rather than an arbitrary lexicographically selected task instance.

## Retry behavior

`retry_exponential_backoff` accepts a numeric multiplier in 3.2.0. `3.5` is a
valid multiplier and `0` disables backoff. Python booleans remain compatible
as `2.0` and `0.0`, but the REST schema is numeric and rejects booleans.

In 3.3.0, tasks can use pluggable retry policies that decide whether and when
to retry based on exceptions or custom backoff logic. Wait failures in
`TriggerDagRunOperator`, including a failed triggered Dag run, participate in
retry-policy handling.

## Python, async, and external task implementations

`PythonOperator` accepts an `async def` function directly in 3.2.0; do not
wrap it in user-managed event-loop code.

`@task.stub` declares a task whose implementation is outside Python. In 3.3.0,
the experimental Coordinator layer can route a stub by queue:

- `JavaCoordinator` runs JVM tasks.
- `ExecutableCoordinator` runs native binaries such as Go programs.

Language runtimes use the Execution API for Variables, Connections, and XComs
while authoring and scheduling remain in Python.

Async tasks also gained native async XCom accessors and
`BaseHook.aget_hook()` in 3.3.0. Structured XCom outputs can round-trip as
Pydantic model instances when the output types are registered from the
worker-side Dag.

## Native template rendering

An operator may set `render_template_as_native_obj=True` or `False` in 3.2.0
to override its Dag. The default `None` inherits the Dag-level setting.

## Internal task APIs removed

Do not call internal `TaskInstance.run()`, `.render_templates()`, or
`.get_template_context()` methods or depend on their related private members;
they were removed in 3.2.0. `PriorityWeightStrategy.serialize()` and
`.deserialize()` are also gone.

## Human-in-the-loop tasks

Airflow 3.1.0 introduces `HITLOperator`, `ApprovalOperator`, and
`HITLEntryOperator` in `apache-airflow-providers-standard`. A HITL task defers
while waiting for an authorized UI or API response. Forms can show XCom values
and Dag parameters, and notification helpers can link responders to the
required-action page.

The UI can add, edit, and delete XCom values as of 3.2.0. HITL task details
show complete approval and rejection history, and `AgenticOperator` workflows
can attach HITL review.

In 3.3.0, a waiting HITL task uses the distinct `awaiting_input` state while
parked on the triggerer. `airflow dags test` waits for that input instead of
spinning indefinitely.

## Deadline Alerts

In 3.1.0, Deadline Alerts are experimental and accept only `AsyncCallback`.
A deadline may be relative to Dag-run queued time or logical date, or a fixed
datetime; it supports positive or negative intervals and a notification
callback.

In 3.2.0, the still-experimental feature adds executor-backed `SyncCallback`
and permits a Dag's `deadline` to be a list mixing synchronous and asynchronous
callbacks. A synchronous callback chooses an executor with its `executor`
parameter but cannot use Connections stored in the metadata database.

In 3.3.0, synchronous callbacks may use Connections and Variables. Deadline
alerts also gain names, Variable-resolved intervals, Core API endpoints, and a
`callback_execution_timeout` setting.

## Designated Dag results

In 3.3.0, `@result` can mark a TaskFlow task as the Dag result. A return-value
XCom can also be designated. The Dag-run NDJSON wait API can then return the
Dag-defined result without requiring clients to name an arbitrary task XCom.

## Resumable external jobs

`ResumableJobMixin`, initially integrated with `SparkSubmitOperator` in 3.3.0,
tracks external work so execution can resume after worker failure rather than
starting the job again. Set an operator's `durable` toggle to opt out of
resumable execution.
