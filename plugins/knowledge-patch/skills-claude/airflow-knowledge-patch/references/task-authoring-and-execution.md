# Task Authoring and Execution

## Context, dates, and callbacks

Airflow 3 removes `tomorrow_ds`, `tomorrow_ds_nodash`, `yesterday_ds`,
`yesterday_ds_nodash`, `prev_ds`, `prev_ds_nodash`, `prev_execution_date`,
`prev_execution_date_success`, `next_execution_date`, `next_ds`,
`next_ds_nodash`, and `execution_date` (3.0-upgrade).

For manual runs, `logical_date` is the requested trigger date; it need not
equal the timetable-resolved `data_interval_start` or `data_interval_end`.
Future logical dates are rejected in 3.0.0. Passing `logical_date=None`, or
omitting it for an Asset- or REST-triggered run, creates a run at the current
time with no logical date or data interval. Such tasks omit all three date keys
from context, so inspect and guard `dag_run.logical_date`.

Skipped tasks no longer receive `on_success_callback` in 3.0.0. Teardown tasks
still execute after early Dag-run termination, but cannot use
`TriggerRule.ALWAYS`; choose a rule that retains upstream dependency semantics.

Dag callbacks receive a task instance relevant to the final Dag state instead
of an arbitrary lexicographically selected instance as of 3.2.0.

## XCom behavior and operator links

`ti.xcom_pull(key=...)` is task-scoped by default after the 3.0-upgrade. Name
the producer when reading another task:

```python
value = ti.xcom_pull(task_ids="upstream_task", key="shared_state")
```

`XCom.set()` and `XCom.get()` reject empty keys in 3.1.0. The removed
`enable_xcom_deserialize_support` option means the API server does not
deserialize unknown Python objects just for display; it renders safer
representations. XCom pickling is gone, so use a custom backend for non-native
representations.

Async tasks gain asynchronous XCom accessors in 3.3.0. Structured XCom outputs
can round-trip as Pydantic models when output types are registered from the
worker-side Dag.

The UI no longer executes custom `BaseOperatorLink` code. Since 3.0.0, an
extra link declares `xcom_key`, task code stores the complete URL at that key,
and the task-detail view retrieves it from the XCom backend.

## Task state stores

The 3.3.0 SDK exposes `task_state_store` and `asset_state_store`. Both persist
JSON state and support `get`, `set`, `delete`, and `clear`; task state survives
retries and runs. Configure expiration, retention, `clear_on_success`, row-size
limits, and retention garbage collection. Core and Execution APIs expose the
state, and triggers can access asset state.

The metadata database is the default storage. Set
`[workers] state_store_backend` for a worker-side backend.
`task_state_store.clear()` no longer accepts `all_map_indices`.

## Failure, retry, and trigger rules

Use Dag argument `fail_fast`, not removed `fail_stop`, from 3.0.0 onward. A
task's effective `priority_weight` is capped by available pool slots, so high
weight cannot bypass pool resource limits.

`ALL_DONE_MIN_ONE_SUCCESS`, added in 3.1.0, runs after every upstream task is
done when at least one succeeded; skipped tasks keep normal skip propagation.

In 3.2.0, `retry_exponential_backoff` accepts a numeric multiplier: `3.5`
uses that factor and `0` disables backoff. Python booleans remain compatible
as `2.0` and `0.0`, but the REST schema is numeric and rejects booleans.

Airflow 3.3.0 adds pluggable task retry policies that decide whether and when
to retry for selected exceptions or custom backoff. `TriggerDagRunOperator`
wait failures, including failed triggered Dags, participate in this policy.

## Templates, Python, and SDK access

An operator may override its Dag's native template rendering in 3.2.0 by
setting `render_template_as_native_obj=True` or `False`; `None` inherits the
Dag setting.

`PythonOperator` accepts an `async def` function directly as of 3.2.0. In
3.3.0 async tasks also gain `BaseHook.aget_hook()` and native async XCom
access, avoiding forced synchronous calls.

Other 3.2.0 runtime access additions include Connection creation from URIs,
previous-task-instance retrieval from `RuntimeTaskInstance`, and the
`BaseXcom` export from `airflow.sdk`.

## External and non-Python work

`@task.stub` declares a task implemented outside Python as of 3.2.0. Airflow
3.3.0's experimental Coordinator layer can route those declarations to
`JavaCoordinator` for JVM work or `ExecutableCoordinator` for native binaries
such as Go. Language runtimes access Variables, Connections, and XComs through
the Execution API while authoring and scheduling remain in Python.

Airflow 3.3.0 introduces `ResumableJobMixin`, initially used by
`SparkSubmitOperator`, so external work can resume after worker failure rather
than restart. Set `durable` to opt out of resumable execution.

An external system can manage work through the TaskInstance API added in
3.2.0.

## Human-in-the-loop and agentic tasks

`HITLOperator`, `ApprovalOperator`, and `HITLEntryOperator` arrive in 3.1.0
through `apache-airflow-providers-standard`. They defer while awaiting an
authorized UI or API response, can show Dag parameters and XCom values in
forms, and provide notification helpers linking responders to the action page.

`AgenticOperator` can attach HITL review in 3.2.0. In 3.3.0 parked HITL tasks
enter the distinct `awaiting_input` state on the triggerer; `airflow dags test`
waits for input instead of spinning indefinitely.

## Dag result

Airflow 3.3.0's `@result` marks a TaskFlow task as the Dag result. A
return-value XCom can be marked equivalently. The Dag-run wait API can then
return the designated result without the caller naming an arbitrary task XCom.
