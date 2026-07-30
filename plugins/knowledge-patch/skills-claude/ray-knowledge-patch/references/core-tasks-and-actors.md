# Core Tasks and Actors

## Type-safe actor handles

For type-checker-friendly actors, leave the original class undecorated, wrap
it with `ray.remote()`, and annotate the wrapper and handle with
`ActorClass[T]` and `ActorProxy[T]`. Mark each remote method with
`@ray.method`. This preserves the declared method result through `.remote()`
as `ObjectRef[R]`.

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

## Cooperative actor-task cancellation

`ray.cancel()` is best-effort, and the result depends on whether the task has
started and what kind of actor method is running:

- If an unscheduled actor task is successfully cancelled, `ray.get()` raises
  `TaskCancelledError`.
- A running regular or threaded actor method is not interrupted. It must poll
  `ray.get_runtime_context().is_canceled()` and perform its own cleanup.
- An async actor method receives `asyncio.Task` cancellation when it reaches
  an `await`.
- Do not call `is_canceled()` from an async actor method; it raises
  `RuntimeError`.
- `recursive=True` also targets tracked child tasks and actor tasks.

```python
import time
import ray

@ray.remote
class Worker:
    def run(self):
        while not ray.get_runtime_context().is_canceled():
            time.sleep(0.1)
        return "cleaned up"

worker = Worker.remote()
ref = worker.run.remote()
ray.cancel(ref, recursive=True)
```

Cancellation-aware code should keep polling intervals bounded and put cleanup
on the path taken after cancellation is observed.

## Task-event reporting

Remote functions and actors accept `enable_task_events=False`. It suppresses
the status and profiling events used by the Dashboard and State API.

```python
@ray.remote(enable_task_events=False)
def quiet_task():
    return 1

@ray.remote
class Worker:
    def work(self):
        return 1

worker = Worker.options(enable_task_events=False).remote()
visible = worker.work.options(enable_task_events=True).remote()
```

The setting is not inherited by nested tasks. For actor work, a method-level
setting overrides the actor-level setting, allowing a generally quiet actor
to expose selected calls or the reverse.
