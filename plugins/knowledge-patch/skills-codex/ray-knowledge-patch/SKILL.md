---
name: ray-knowledge-patch
description: Ray
version: null
license: MIT
metadata:
  author: Nevaberry
---


# Ray Knowledge Patch

Use this skill when working with Ray Core, Ray Data, Ray Train, Ray Tune,
Ray Serve, or KubeRay. It captures API boundaries, scheduling behavior,
recovery limits, and lifecycle rules that are easy to miss when these products
are combined.

## Reference index

| Reference | Topics |
| --- | --- |
| [references/tasks-and-actors.md](references/tasks-and-actors.md) | Typed actor handles, cancellation, task-event reporting |
| [references/data-processing.md](references/data-processing.md) | Streaming, batches, compute pools, ordering, placement groups, async transforms, expressions |
| [references/training-and-api-stability.md](references/training-and-api-stability.md) | Train V2, dataset splitting, subclusters, preprocessors, backpressure, API stability |
| [references/tuning.md](references/tuning.md) | Result emission, time budgets, scheduler constraints, dynamic resources |
| [references/serving-and-recovery.md](references/serving-and-recovery.md) | Deployment responses, replica and controller recovery, REST management |
| [references/kubernetes-operations.md](references/kubernetes-operations.md) | RayCluster, RayJob modes and lifecycle, cleanup, RayService endpoints |

## Migration and deprecation checks

### Opt in deliberately to Train V2

Starting in Ray 2.43, Train V2 is enabled with an environment variable:

```bash
RAY_TRAIN_V2_ENABLED=1 python train.py
```

Do not assume imports from the deprecated V1 reference carry over. Check the
V2 API when migrating framework helpers, trainers, configuration paths, and
session utilities.

### Respect API stability windows

An API without an annotation is a Developer API and may change. Public and
deprecated APIs require explicit annotations.

For a Stable API demotion or parameter change:

- warn users;
- keep old and new parameters together during the transition; and
- use a deadline of six months or 25 minor versions.

For Beta APIs, the deadline is three months or 12 minor versions. Alpha APIs
have no stability guarantee.

### Treat RayJob cleanup mechanisms as alternatives

KubeRay 1.6.0 provides beta `deletionStrategy` rules behind the
`RayJobDeletionPolicy` feature gate. Do not combine rules-based cleanup with
`shutdownAfterJobFinishes` or the global `ttlSecondsAfterFinished`.
The older `onSuccess` and `onFailure` style is deprecated.

`shutdownAfterJobFinishes` defaults to false. A
`ttlSecondsAfterFinished` value only takes effect when shutdown is enabled.
The operator can also delete the RayJob custom resource and everything it
created when `DELETE_RAYJOB_CR_AFTER_JOB_FINISHES=true` and shutdown is
enabled.

### Check RayJob submission-mode constraints

`K8sJobMode` is the default. A `submitterPodTemplate` applies only in that
mode. `SidecarMode` requires the head Pod restart policy to be `Never` and
does not support:

- `clusterSelector`;
- `submitterPodTemplate`; or
- `submitterConfig`.

`InteractiveMode` is alpha. See the Kubernetes reference before changing
submission mode, retries, deadlines, suspension, or cleanup.

## Core task and actor quick reference

### Keep actor types through `.remote()`

Leave the original class undecorated, mark its remote methods with
`@ray.method`, then wrap it with `ray.remote()`. Annotate the wrapper with
`ActorClass[T]` and the handle with `ActorProxy[T]`; method calls then retain
`ObjectRef[R]` return types.

```python
import ray
from ray.actor import ActorClass, ActorProxy

class Counter:
    @ray.method
    def increment(self) -> int:
        return 1

CounterActor: ActorClass[Counter] = ray.remote(Counter)
counter: ActorProxy[Counter] = CounterActor.remote()
result: ray.ObjectRef[int] = counter.increment.remote()
```

### Implement cooperative cancellation

`ray.cancel()` is best-effort. An unscheduled actor task that is cancelled
successfully raises `TaskCancelledError` from `ray.get()`. Running regular or
threaded actor methods are not interrupted; poll:

```python
ray.get_runtime_context().is_canceled()
```

Async actor methods instead receive `asyncio.Task` cancellation at an
`await`. Calling `is_canceled()` from an async actor method raises
`RuntimeError`. With `recursive=True`, cancellation also targets tracked
child and actor tasks.

### Control task-event visibility locally

Set `enable_task_events=False` on a remote function or actor to suppress the
status and profiling events consumed by the Dashboard and State API.
Nested tasks do not inherit the parent setting. An actor method setting
overrides the actor setting.

## Ray Data quick reference

### Identify streaming barriers

Dataset transforms are lazy. Non-shuffle operators can execute concurrently
as a streaming pipeline after consumption starts. `sort()` and `groupby()`
must materialize data, so streaming pauses until their shuffle completes.

### Choose transform execution correctly

- Functions run as tasks. Cap concurrency with `TaskPoolStrategy(size=n)`.
- Callable classes run as actors, initialize once per worker, and default to
  an autoscaling actor pool.
- Use `ActorPoolStrategy` when the actor pool must be fixed.
- `memory`, `num_cpus`, and `num_gpus` affect scheduling; they do not enforce
  physical resource limits.
- Async transforms must be callable classes with `async def __call__`.
  Function-based async transforms are unsupported, and the feature requires
  `uvloop==0.21.0`.

`map_batches(batch_size="auto")` enables automatic sizing, but GPU transforms
need an explicit integer batch size. Lower it when a worker runs out of memory.

### Make ordering explicit

Transforms do not preserve block order by default. A sorted Dataset retains
order; otherwise enable `preserve_order` in the current Data context when the
result requires it. Expect reduced performance when workers finish unevenly.

## Ray Train quick reference

### Split only the intended datasets

Train normally calls `Dataset.streaming_split()` so every worker receives a
disjoint shard of every Dataset. Set
`DataConfig(datasets_to_split=[...])` to select which datasets are sharded.
An unlisted validation Dataset is complete on every worker; metrics from a
split validation Dataset must be aggregated across workers.

### Apply subcluster selectors twice

To pin Train data work to a labeled subcluster:

1. Copy the current `DataContext`, set its `label_selector`, and use it while
   constructing the Dataset. This covers file listing and schema work.
2. Repeat the selector in `DataConfig.execution_options` for ingestion.

Train replaces the Dataset context's execution options, so the
construction-time selector alone does not pin per-worker ingest.

### Preserve preprocessing and control backpressure

Fit and apply preprocessing before constructing the Trainer. Serialize the
preprocessor, encode the bytes for JSON-compatible Trainer `metadata`, and
restore it from checkpoint metadata.

Set each Dataset's execution-context `resource_limits` with an
`ExecutionResources(object_store_memory=...)` limit. Ray Data slows
production at the limit so training consumers are not overrun by spilled
data.

## Tune quick reference

- A function trainable may call `tune.report()` for intermediate metrics,
  return one dictionary for its final result, or yield successive result
  dictionaries.
- Do not call `tune.report()` inside a class-based `Trainable`.
- Combine `num_samples=-1` with `time_budget_s` for open-ended trial
  generation bounded by wall-clock time.
- A finite `num_samples` caps the number of trials.
- `ResourceChangingScheduler` can wrap another scheduler and change trial
  resources while tuning runs.

Checkpointing and search-algorithm compatibility differ among ASHA, Median
Stopping, HyperBand, BOHB, PBT, and PB2. Consult the tuning reference before
combining them.

## Serve quick reference

A deployment-handle method returns `DeploymentResponse` immediately. Await
it for the value, or pass it directly to another deployment-handle call to
pipeline intermediate results without materializing them locally.

Application exceptions return HTTP 500 with traceback information but do not
kill the replica. Serve replaces failed replicas and restarts failed proxies
and the controller. Controller state is restored from the GCS, but transient
connections and internal request queues are lost.

HTTP, gRPC, and deployment-handle traffic can continue while the controller
is unavailable. Autoscaling pauses and resumes after recovery without the
metrics gathered before the failure. Whole-cluster recovery belongs at the
KubeRay layer.

## KubeRay quick reference

A RayJob can embed `rayClusterSpec` to create a cluster or use
`clusterSelector` to target an existing RayCluster. KubeRay submits its
`entrypoint` only after the cluster is ready.

Keep retry scopes separate:

- top-level `backoffLimit` defaults to zero and creates a new RayCluster for
  each retry;
- `submitterConfig.backoffLimit` defaults to two and retries the submitter
  Kubernetes Job.

`preRunningDeadlineSeconds` bounds the wait for `Running`; zero disables it.
`activeDeadlineSeconds` bounds the time to reach `Complete` or `Failed`.
Do not manually change `suspend` when Kueue owns RayJob scheduling.

RayService manages a RayCluster and Ray Serve applications.
`serveConfigV2` accepts multi-application configuration. Once Serve has
endpoints, RayService reports `Ready=True`, exposes Dashboard access on its
head service at port 8265, and exposes application HTTP traffic on its Serve
service at port 8000.
