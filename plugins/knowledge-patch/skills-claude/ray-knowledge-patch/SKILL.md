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
Ray Serve, or KubeRay, especially for API migration, execution semantics,
resource scheduling, cancellation, failure recovery, and Kubernetes lifecycle
decisions.

## Reference index

| Reference | Topics |
| --- | --- |
| [Core tasks and actors](references/core-tasks-and-actors.md) | Typed actor handles, cooperative cancellation, task-event reporting |
| [Data execution](references/data-execution.md) | Streaming barriers, batch sizing, transform pools, ordering, placement groups, async transforms, expressions |
| [Train and API lifecycle](references/train-and-api-lifecycle.md) | Train V2, dataset sharding, subclusters, checkpoint metadata, backpressure, API stability |
| [Tune scheduling](references/tune-scheduling.md) | Function results, open-ended sampling, scheduler compatibility, dynamic resources |
| [Serve runtime and recovery](references/serve-runtime-and-recovery.md) | Response pipelining, replica and controller failures, management endpoints |
| [KubeRay jobs and services](references/kuberay-jobs-and-services.md) | RayCluster bootstrap, RayJob modes, retries, deadlines, cleanup, RayService |

## Migration and deprecation priorities

### Opt in to Train V2 deliberately

Starting in Ray 2.43, enable Train V2 before relying on its implementation and
APIs:

```bash
RAY_TRAIN_V2_ENABLED=1 python train.py
```

The separate V1 reference is deprecated. It includes legacy trainers,
framework helpers, configuration paths, and session utilities. Check the V2
API during migration instead of carrying old imports forward unchanged.

### Follow the current RayJob cleanup model

KubeRay 1.6.0 adds beta `deletionStrategy` rules behind the
`RayJobDeletionPolicy` feature gate. Rules may select `DeleteWorkers`,
`DeleteCluster`, `DeleteSelf`, or `DeleteNone` independently for `SUCCEEDED`
and `FAILED`, with optional per-rule TTLs.

Rules-based cleanup cannot be combined with `shutdownAfterJobFinishes` or the
global `ttlSecondsAfterFinished`. The older `onSuccess` and `onFailure` style
is deprecated.

### Respect API stability annotations

An unannotated Ray API is a Developer API by default. Public and deprecated
APIs require explicit annotations. Stable demotions or parameter changes
require warnings, old/new parameter coexistence, and a six-month or
25-minor-version transition. Beta uses three months or 12 minor versions;
Alpha has no stability guarantee.

## Core execution quick reference

### Type actor handles without decorating the source class

Keep the class undecorated, mark remote methods with `@ray.method`, wrap the
class with `ray.remote()`, then use `ActorClass[T]` and `ActorProxy[T]`. This
preserves a method result `R` as `ObjectRef[R]`.

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

### Treat cancellation as cooperative

`ray.cancel()` is best-effort. A successfully cancelled unscheduled actor task
causes `ray.get()` to raise `TaskCancelledError`. A running regular or
threaded actor method is not interrupted; poll
`ray.get_runtime_context().is_canceled()` and clean up explicitly.

Async actor methods instead receive `asyncio.Task` cancellation at an
`await`; calling `is_canceled()` in an async actor method raises
`RuntimeError`. Set `recursive=True` when tracked child and actor tasks should
also be targeted.

### Control event visibility at the right level

Set `enable_task_events=False` on a remote function or actor to suppress the
status and profiling events consumed by the Dashboard and State API. Nested
tasks do not inherit this setting. An actor method setting overrides its
actor's setting.

## Ray Data quick reference

### Account for materialization barriers

Dataset transformations are lazy. Once consumption starts, non-shuffle
operators can overlap as a streaming pipeline. `sort()` and `groupby()` must
materialize data; streaming pauses until their shuffle completes.

### Match the transform form to the worker model

- Function transforms run as tasks; cap concurrency with
  `TaskPoolStrategy(size=n)`.
- Callable classes run as actors, initialize once per worker, and default to
  an autoscaling actor pool. Use `ActorPoolStrategy(size=n)` for a fixed pool.
- `memory`, `num_cpus`, and `num_gpus` are logical scheduling requirements,
  not enforced physical limits.
- Async transforms must be callable classes with `async def __call__`;
  function transforms are unsupported and the feature requires
  `uvloop==0.21.0`.

`map_batches(batch_size="auto")` is supported for automatic sizing, but a GPU
transform requires an explicit integer. Reduce that integer when a worker
runs out of memory.

### Ask explicitly for row order

Transforms do not preserve block order by default. A sorted Dataset does, or
set `DataContext.get_current().execution_options.preserve_order = True`.
Preserving order can reduce throughput when workers complete unevenly.

## Train quick reference

### Split only the datasets that need sharding

Train normally calls `Dataset.streaming_split()` so each worker gets a
disjoint shard of every Dataset. Set
`DataConfig(datasets_to_split=["train"])` to leave an unlisted validation
Dataset complete on every worker. Aggregate validation results across workers
when validation is split.

### Apply subcluster selectors twice

For a labeled data subcluster, use a copied `DataContext` while constructing
the Dataset to pin listing and schema work. Repeat the selector in
`DataConfig.execution_options` to pin ingestion. Train replaces the Dataset
context's execution options, so the construction-time selector alone is not
enough.

### Carry fitted preprocessing in checkpoint metadata

Fit and apply preprocessing before Trainer construction. Serialize the fitted
preprocessor, base64-encode its bytes for the JSON-compatible Trainer
`metadata`, and deserialize the payload from checkpoint metadata later.
Trainer metadata is available through `TrainContext.get_metadata()` and is
attached to checkpoints saved by the Trainer.

## Tune quick reference

Function trainables may call `tune.report()` for intermediate results, return
a dictionary for only the final result, or yield successive result
dictionaries. Do not call `tune.report()` inside a class-based `Trainable`.

For trials generated until a wall-clock limit, combine `num_samples=-1` with
`time_budget_s`. A finite `num_samples` remains a hard trial-count cap.

| Scheduler | Checkpointing | Search algorithm compatibility |
| --- | --- | --- |
| ASHA, Median Stopping | Must be off | Compatible |
| HyperBand | Required | Compatible |
| BOHB | Required | Only `TuneBOHB` |
| PBT, PB2 | Required | Incompatible |

Wrap another scheduler with `ResourceChangingScheduler` to change a trial's
resource requirements while tuning is running.

## Serve quick reference

A deployment call returns `DeploymentResponse` immediately. Await it for a
local value or pass it directly to another `DeploymentHandle` call to pipeline
composed deployments without materializing the intermediate result locally.

Application exceptions return HTTP 500 with traceback information and do not
kill the replica. Serve replaces failed replicas, proxies, and the controller,
and restores routing policies and deployment configuration from the GCS.
Transient connections and internal request queues are not restored; complete
cluster recovery belongs at the KubeRay layer.

HTTP, gRPC, and deployment-handle traffic may continue during controller
failure. Autoscaling pauses and resumes after recovery, without the metrics
collected before the failure.

## KubeRay quick reference

- `K8sJobMode` is the default RayJob submission mode and creates a submitter
  Kubernetes Job.
- `HTTPMode` submits from the operator; alpha `InteractiveMode` waits for user
  submission; `SidecarMode` injects the submitter into the head Pod.
- Sidecar mode requires head Pod restart policy `Never` and does not support
  `clusterSelector`, `submitterPodTemplate`, or `submitterConfig`.
- Top-level `backoffLimit` retries with a new RayCluster and defaults to zero;
  `submitterConfig.backoffLimit` retries only the submitter Job and defaults
  to two.
- `preRunningDeadlineSeconds` bounds reaching `Running`; zero disables it.
  `activeDeadlineSeconds` bounds reaching `Complete` or `Failed`.
- `shutdownAfterJobFinishes` defaults to false.
  `ttlSecondsAfterFinished` applies only when shutdown is enabled.

Use the linked references for complete configuration details and lifecycle
interactions before changing production scheduling, cleanup, or recovery
behavior.
