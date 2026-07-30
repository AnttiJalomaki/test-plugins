# Tasks and actors

## Type-safe actor handles

For type-checker-friendly actors, keep the implementation class undecorated
and wrap it after definition. Mark remotely callable methods with
`@ray.method`, annotate the wrapper as `ActorClass[T]`, and annotate the
handle as `ActorProxy[T]`. The remote method's declared result then becomes
`ObjectRef[R]`.

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

`ray.cancel()` is best-effort, and the result depends on the task state and
actor execution model:

- If an unscheduled actor task is cancelled successfully, `ray.get()` raises
  `TaskCancelledError`.
- A running regular or threaded actor method is not interrupted. It must poll
  `ray.get_runtime_context().is_canceled()` and stop cooperatively.
- An async actor method receives `asyncio.Task` cancellation when it reaches
  an `await`.
- Calling `is_canceled()` from an async actor method raises `RuntimeError`.
- `recursive=True` also targets tracked child and actor tasks.

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

## Task-event reporting

Remote functions and actors accept `enable_task_events=False`. This suppresses
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

The setting is not implicitly propagated through the call graph:

- a nested task does not inherit its parent task's setting; and
- a setting on an actor method overrides the setting on its actor.

Set the option explicitly at each boundary whose events should be hidden or
shown.
