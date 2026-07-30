# Execution and Operations

Use this reference for run coordination, concurrency, executors, backfills,
daemon behavior, GraphQL clients, retries, and operational limits.

## Run coordination and pools

The queued run coordinator is the default as of 1.10.0. The Dagster daemon
must be running for queued runs to launch. To retain immediate in-process
launching, configure:

```yaml
run_coordinator:
  module: dagster.core.run_coordinator.sync_in_memory_run_coordinator
  class: SyncInMemoryRunCoordinator
```

Run blocking for concurrency keys and pools is enabled by default. With op
granularity, the coordinator dequeues a run once at least one op can execute.
With run granularity, every pool the run uses must have a free slot.

Pool-name validation evolved. In 1.10.0, names allowed only letters, numbers,
dashes, and underscores. In 1.12.0, names were relaxed to allow any
non-whitespace character, replacing both the original restriction and the
intermediate slash allowance.

The `dagster-dbt`, `dagster-dlt`, and `dagster-sling` integrations support
pools as of 1.10.0.

## Step dependency behavior

Every executor supports `step_dependency_config.require_upstream_step_success`
as of 1.11.0. Set it to `false` to start a downstream step after its required
outputs are available, even if the producing multi-asset step is still
running:

```json
{"step_dependency_config": {"require_upstream_step_success": false}}
```

## Custom executor failure handling

Since 1.12-upgrade, process-based executors can recover from resource
initialization failures through step retries. A custom `Executor` must emit and
register an explicit failure-or-retry event for every such failure; otherwise
the run may remain in `started` status.

```python
if event.is_resource_init_failure:
    failure_or_retry_event = self.get_failure_or_retry_event_after_error(
        step_context,
        event.engine_event_data.error,
        active_execution.get_known_state(),
    )
    yield failure_or_retry_event
    active_execution.handle_event(failure_or_retry_event)
```

## Backfills and run configuration

`BackfillPolicy` is GA as of 1.11.0. Backfill submission uses a thread pool
with four daemon workers by default. Failed backfills cancel their in-progress
runs before terminating, and asset backfills can receive run config.

As of 1.13.0, job backfills retry transient daemon failures.
`DAGSTER_MAX_ASSET_BACKFILL_RETRIES` was renamed to
`DAGSTER_MAX_BACKFILL_RETRIES`; the old name remains a fallback.

Partial run configuration now inherits job-level defaults for omitted sections
as of 1.13.0.

## Daemon dispatch and failure context

Schedule, sensor, and asset-daemon ticks dispatch instigators round-robin
across code locations as of 1.13.0. When a run fails because a step failed,
the originating step error is available on the run-failure sensor context.

## GraphQL clients and pagination

`DagsterGraphQLClient.submit_job_execution` accepts `asset_selection` as of
1.11.0.

`logsForRun` and `eventConnection` return at most 1,000 events per query by
default. Follow the returned cursor until all required events have been read;
do not assume one response is complete.

As of 1.13.0, `DagsterGraphQLClient` accepts `path_prefix` for webservers
mounted below the URL root. The GraphQL `Run` type adds an optional selection
`limit`, `assetSelectionCount`, and `assetCheckSelectionCount`, so a client can
show a bounded preview while retaining true totals.

## Errors, logs, and heartbeat limits

Event error messages and stack traces over 500 KB are truncated by default
(1.11.0). Override the threshold with
`DAGSTER_EVENT_ERROR_FIELD_SIZE_LIMIT`.

`DAGSTER_GRPC_PROXY_HEARTBEAT_TTL_SECONDS` configures the proxy gRPC heartbeat
TTL; its default is 30 seconds (1.11.0).

The SQLite event-log `busy_timeout` default increased from 5 to 30 seconds in
1.13.0.

## Cleanup and transient failures

The Kubernetes executor option `enable_owner_references` ties step jobs and
pods to the run pod for garbage collection (1.11.0).

In 1.13.0, ECS stops caused by `InsufficientFreeAddressesInSubnet` or
`Task provisioning failed` are classified as transient. Dagster retries the
affected run instead of permanently failing it.

