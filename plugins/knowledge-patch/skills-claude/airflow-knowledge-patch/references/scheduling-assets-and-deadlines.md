# Scheduling, Assets, and Deadlines

## Defaults, intervals, and run creation

The Airflow 3.0-upgrade changes both `catchup_by_default` and
`create_cron_data_intervals` to `False`. A bare cron `schedule=` therefore
uses `CronTriggerTimetable`, not `CronDataIntervalTimetable`. Set
`create_cron_data_intervals=True` before upgrading when tasks depend on
interval boundaries or derived `ds`/`ts` values. Reverting after Airflow 3
runs already exist skips one scheduled run to avoid duplicating a logical date.

Event-driven runs can have no logical date or data interval in 3.0.0. Asset
and REST triggers that omit `logical_date` preserve `None`; guard scheduling
logic accordingly. A continuous Dag may omit `start_date` as of 3.2.0 and
starts immediately when unpaused.

## Backfills, clearing, and Dag versions

Since 3.0.0 the scheduler manages backfills like ordinary Dag runs, including
versioning and observability. Start and monitor them through REST or the UI
rather than as separate CLI jobs.

Dag structures are versioned in metadata, so historical task renames and
dependency changes are visible through API and UI. The triggerer does not
initialize Dag bundles; trigger implementations must be importable elsewhere
on `sys.path`, not solely from a bundle.

In 3.2.0, Dag clear accepts `only_new` to target newly added task instances.
In 3.3.0, `rerun_with_latest_version` chooses the original or latest Dag bundle
for clear, rerun, backfill, and `TriggerDagRunOperator` reruns. Precedence is:
request parameter or CLI flag, Dag setting, `[core] rerun_with_latest_version`,
then `False` for clear/rerun and `True` for backfill.

## Asset event addressing and aliases

From 3.0.0, event maps reject string keys. Address `inlet_events`,
`outlet_events`, and `triggering_asset_events` with `Asset`, `AssetAlias`, or
`Asset.ref`, or use their lookup helpers:

```python
outlet_events[Asset.ref(name="myasset")]
outlet_events[AssetAlias(name="myalias")]
outlet_events.for_asset(name="myasset")
outlet_events.for_asset_alias(name="myalias")
```

`create_asset_aliases()` creates aliases shared across Dag files. Asset API
responses use `scheduled_dags`, not `consuming_dags`, as of 3.1.0; the field
means Dags that put the Asset in `schedule`, not all Dags that use it.

## Partition-aware Assets

Airflow 3.2.0 can trigger downstream Dags only for updated partitions. Use
`AllowedKeyMapper` to validate keys and `ChainMapper` to compose mappings.
Temporal mapper names use `StartOfXXXMapper`, not `ToXXXMapper`. Inlet events
can be lazily filtered by time, ordering, and limit, and listeners can receive
Asset-emission events.

Airflow 3.3.0 adds `RollupMapper`, `FanOutMapper`, categorical
`FixedKeyMapper`, and `SegmentWindow`. Mappings can use temporal windows,
`WaitForAll` or `MinimumCount(n)`, and forward or backward fan-out.
`PartitionedAtRuntime` assigns keys when a Dag run starts. Bound expansion with
`[scheduler] partition_mapper_max_downstream_keys` or a per-mapper override.

REST adds `clearPartitions` and bulk `/dags/{dag_id}/clearDagRuns` with
`partition_key` and `partition_date` window selectors in 3.3.0. CLI clearing
and backfills accept partition ranges. Producer partition dates and
task-emitted keys propagate through Asset events to partitioned consumers.

## Trigger and pool scheduling

Pool capacity caps effective `priority_weight` in 3.0.0. In 3.2.0,
`airflow scheduler --only-idle` makes `--num-runs` count only idle scheduler
loops, allowing queued tasks and triggered Dags to finish before exit:

```bash
airflow scheduler --num-runs 1 --only-idle
```

The trigger command gains `--queues` in 3.2.0 to route triggers by task queue
to Triggerer hosts. `max_trigger_to_select_per_loop` bounds each selection
loop in high-availability Triggerer deployments. In 3.3.0, triggerers can also
be assigned and filtered by team.

## Multi-team deployments

Airflow 3.2.0 introduces experimental isolation for each team's Dags,
Connections, Variables, pools, executors, resources, and permissions.

In 3.3.0, the Asset SDK replaces `allow_producer_teams` with `access_control`;
`AssetAccessControl` adds `consumer_teams` and `allow_global`. Asset queries and
the XCom Execution API are team-scoped. Pool scheduling enforces team
ownership, and pool CLI commands accept `--team-name`. Assets, pools, and
triggers must therefore participate in the same team model.

## Deadline Alerts

Deadline Alerts are the Airflow 3 replacement for SLAs. In 3.1.0 the feature
is experimental and accepts only `AsyncCallback`. A deadline can be relative
to queued time, logical date, or a fixed datetime, with a positive or negative
interval and notification callback.

Airflow 3.2.0 adds executor-run `SyncCallback`, whose `executor` parameter
selects an executor, and permits a Dag's `deadline` to be a list mixing sync
and async callbacks. The feature remains experimental. At this version a
synchronous callback cannot use Connections stored in the metadata database.

Airflow 3.3.0 allows synchronous deadline callbacks to use runtime Connections
and Variables. It also adds deadline names, Variable-resolved intervals, Core
API endpoints, and `callback_execution_timeout`.

## Trigger and termination semantics

`ALL_DONE_MIN_ONE_SUCCESS` in 3.1.0 waits for all upstream work and requires at
least one success. Teardown tasks continue after Dag termination but cannot use
`TriggerRule.ALWAYS`. Failed waits in `TriggerDagRunOperator`, including a
failed triggered Dag, participate in pluggable retry policies in 3.3.0.
