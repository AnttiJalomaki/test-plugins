# Queues, Concurrency, Scheduling, and Processes

## Contents

- [Dispatch, routing, and job policy](#dispatch-routing-and-job-policy)
- [Batches, uniqueness, and retry policy](#batches-uniqueness-and-retry-policy)
- [Queue drivers, storage, and failover](#queue-drivers-storage-and-failover)
- [Deferred, background, and concurrent execution](#deferred-background-and-concurrent-execution)
- [Worker lifecycle and control](#worker-lifecycle-and-control)
- [Scheduling](#scheduling)
- [Queue and process diagnostics](#queue-and-process-diagnostics)

## Dispatch, routing, and job policy

- **Key-preserving concurrency** `[12.0-upgrade]`: associative input passed to
  `Concurrency::run()` returns associative keyed results.
- **Multiple chain additions** `[2025-05]`: `appendToChain()` and
  `prependToChain()` accept arrays of jobs.
- **Batchable generation** `[2025-06]`: scaffold batch support with
  `php artisan make:job ImportChunk --batchable`.
- **Conditional job lists** `[2026-01]`: `Bus::batch()` and `Bus::chain()`
  discard falsy members, allowing conditional jobs without prefiltering.
- **Deferred sync callbacks** `[2026-02]`: deferred callbacks registered while
  using the sync queue are retained.
- **Central queue routes** `[13.0.0]`: `Queue::route(Job::class, connection: ...,
  queue: ...)` centrally selects a job class's defaults.
- **Single-string queue routes** `[2026-04]`: a string route value is the queue
  name, not the connection name.
- **Delay precedence** `[2026-04]`: the `Delay` attribute applies to bus
  dispatch, queued notifications, and queued mailables; runtime `onQueue()`
  wins over a class-level queue attribute.
- **Debounced jobs** `[2026-04]`: queued jobs support debouncing, and
  `DebounceFor` declarations are inherited by subclasses.
- **Preparing dispatch** `[2026-05]`: implement `PreparesForDispatch` to run a
  preparation step before dispatch.
- **Queue attribute refinements** `[2026-06]`: time attributes and
  `DebounceFor` accept explicit units; traits may declare queue attributes;
  child job properties override inherited attributes.
- **Batch and bulk delays** `[2026-07]`: `Delay` is honored by `Bus::batch()`
  and bulk dispatch.

## Batches, uniqueness, and retry policy

- **Unique locks after rollback** `[2025-04]`: an after-commit
  `ShouldBeUnique` job discarded by transaction rollback releases its unique
  lock.
- **Conditional throttled failure** `[2025-07]`:
  `ThrottlesExceptions::failWhen()` fails the job when its callback matches the
  exception instead of continuing throttled retries.
- **Batch member failure callbacks** `[2025-09]`: jobs running in a batch can
  define their own failure cleanup or reporting callbacks.
- **Dynamic listener tries** `[2025-09]`: queueable listeners may calculate
  retry count with `tries()`.
- **Unique queued listeners** `[2026-02]`: queued event listeners can use the
  queue uniqueness mechanism.
- **Batch cancellation event** `[2026-02]`: cancelling a batch dispatches
  `BatchCancelled`.
- **Missing models in notifications** `[13.0-upgrade]`: queued notifications
  honor `#[DeleteWhenMissingModels]` and `$deleteWhenMissingModels`.
- **Dynamic missing models for listeners** `[13.0.0]`: queued listeners may
  decide at runtime whether to delete a job whose serialized model is missing.

## Queue drivers, storage, and failover

- **Pheanstalk 7** `[2025-04]`: the Beanstalkd integration supports
  `pda/pheanstalk` 7.
- **Standardized sizing** `[2025-06]`: queue `size()` behavior is standardized,
  and extended metrics are available across queue integrations.
- **SQS fair queues** `[2025-09]`: fair queues and FIFO queues support message
  groups; use the final `messageGroup` property, not the early `group` name.
- **Failover queues** `[2025-10]`: queue connections can fail over to
  alternatives; `QueueFailedOver` contains the originating exception.
- **Explicit Redis middleware connection** `[2026-02]`: Redis-backed queue
  middleware can select a non-default Redis connection.
- **Laravel 13 queue metrics contract** `[13.0-upgrade]`: custom drivers must
  implement `pendingSize`, `delayedSize`, `reservedSize`, and
  `creationTimeOfOldestPendingJob`.
- **Beanstalkd compatibility** `[13.0.0]`: Laravel 13 supports Pheanstalk 8.x
  and no longer supports 5.x.
- **Redis Cluster queues** `[2026-04]`: queues and `ConcurrencyLimiter` support
  Redis Cluster directly.
- **Named SQS credential providers** `[2026-04]`: Laravel 12 and 13 SQS
  connections accept named credential providers.
- **Wider attempt counters** `[2026-04]`: custom job-table migrations should
  store `attempts` as a small integer rather than a tiny integer.
- **Disk-backed SQS overflow** `[2026-05]`: optionally offload oversized SQS
  payloads to a disk; `queue:clear` can also flush the overflow store.
- **Laravel Cloud queues** `[2026-05]`: Laravel 12 and 13 provide a Cloud queue
  driver, Cloud metrics with `after_commit`, cached configuration, and scoped
  filesystem support. Managed queues boot before application providers;
  missing queues throw `ManagedQueueNotFoundException`; `Cloud-Request-ID` is
  logged for correlation.

## Deferred, background, and concurrent execution

- **Queue memory zero** `[12.0.0]`: `queue:work --memory=0` disables the memory
  exceeded check.
- **Deferred queue execution** `[2025-10]`: use the deferred queue path when
  work can execute after the response without an external worker.
- **Background queue execution** `[2025-11]`: `Concurrently::defer()` provides a
  background processing path distinct from the deferred queue.
- **Concurrency timeouts** `[2026-05]`: `Concurrency::run()` accepts runtime
  timeouts.
- **Process timeout intervals** `[13.0.0]`: pending process timeouts accept
  `CarbonInterval`.
- **Macroable invoked processes** `[2026-06]`: `InvokedProcess` supports
  application macros.

## Worker lifecycle and control

- **Worker startup** `[2025-06]`: `WorkerStarting` fires once when a queue worker
  daemon starts.
- **Memory-limit exit codes** `[2025-09]`: configure the exit code used when a
  worker exceeds its memory limit.
- **Queue pause and resume** `[2025-11]`: pause queues indefinitely or for a
  number of seconds, then resume them.
- **Queue lifecycle events** `[2026-01]`: pause and resume emit events; sync
  jobs dispatch `JobAttempted`; `JobPopping` includes the queue;
  `JobReleasedAfterException` includes backoff; `BatchFinished` marks batch
  completion.
- **`JobAttempted` exception data** `[13.0-upgrade]`: read the exception or
  `null` from `$exception`; `$exceptionOccurred` is removed.
- **`QueueBusy` rename** `[13.0-upgrade]`: use `$connectionName`, not
  `$connection`.
- **Job inspection** `[2026-04]`: inspect queued jobs through framework queue
  APIs instead of backend internals.
- **Worker signals and batches** `[2026-04]`: `BatchStarted` reports startup;
  jobs can react to worker signals; `WorkerInterrupted` and `WorkerStopReason`
  report interruptions and lost connections; managed workers disable pausing.
- **Expanded worker events** `[2026-05]`: `WorkerIdle` is available;
  `WorkerPausing`, `WorkerResuming`, `WorkerInterrupted`, and `Looping` include
  `WorkerOptions`. Override timeout exit codes or opt out of restart on lost
  connection.
- **Richer inspection** `[2026-06]`: `InspectedJob` includes payload and queue
  information.
- **Worker stopping telemetry** `[2026-06]`: `WorkerStopping` includes processed
  count and last-job time, which is `null` if no job ran.
- **More stopping data** `[2026-07]`: `WorkerStopping` also includes memory
  usage.
- **Released-job exception** `[2026-07]`: `JobReleasedAfterException` exposes
  the causing exception.

## Scheduling

- **Email output default** `[12.0.0]`: `emailOutput()` sends only when output
  exists by default.
- **Foreground failures** `[2025-06]`: failed foreground scheduled tasks
  dispatch `ScheduledTaskFailed`, matching background failure observation.
- **Scheduler output modes** `[2025-09]`: `schedule:work --whisper` reduces
  output; `schedule:list --json` is machine-readable.
- **Enum scheduler store** `[2025-10]`: `Schedule::useCache()` accepts an enum
  cache-store selector.
- **Schedule-list environments** `[2025-11]`: JSON schedule listings include
  each event's environment.
- **Day-of-month schedules** `[2025-11]`: `daysOfMonth([1, 15])` targets exact
  calendar days.
- **Schedule group macros** `[2026-02]`: macros on scheduled command events can
  be applied within schedule groups.
- **Deferred schedule registration** `[13.0-upgrade]`: callbacks passed to
  `ApplicationBuilder::withScheduling()` register only when `Schedule` is
  resolved.
- **Pause and resume commands** `[13.0.0]`: use `schedule:pause` and
  `schedule:resume`; lifecycle events accompany both.
- **Termination handling in groups** `[2026-04]`:
  `releaseOnTerminationSignals()` propagates from a schedule group to its
  events.
- **Environment filters** `[2026-05]`: filter `schedule:list` by environment.
- **Group callbacks** `[2026-05]`: `Schedule::group()` accepts lifecycle and
  output callbacks; scheduled callback parameters resolve by type and may
  receive the `Event`.
- **Scheduler cache opt-outs** `[2026-06]`: disable pause and interrupt cache
  checks where shared-cache controls are unwanted.

## Queue and process diagnostics

- **Oldest pending metric** `[2026-02]`: `queue:monitor` reports
  `oldest_pending`.
- **Failed jobs as JSON** `[2026-05]`: `queue:failed --json` is
  machine-readable; normal output reports the real job class.
- **Clearing multiple queues** `[2026-07]`: `queue:clear` accepts multiple
  queues in one invocation.
