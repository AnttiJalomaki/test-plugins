# Ray Data Execution

## Lazy execution and shuffle barriers

Dataset transformations are lazy. When a Dataset is consumed, non-shuffle
operators can execute concurrently as a streaming pipeline. `sort()` and
`groupby()` require materialization, so streaming stops until their shuffle
finishes. Plan these operations as execution barriers rather than expecting
records to flow continuously through them.

## Batch sizing

`map_batches()` accepts `batch_size="auto"` for automatic sizing. GPU
transforms are the exception: they require an explicit integer batch size.
Reduce that integer if the transform worker runs out of memory.

```python
ds = ds.map_batches(
    GpuPredictor,
    num_gpus=1,
    batch_size=64,
)
```

## Transformation pools and logical resources

A function transform uses Ray tasks. Bound its task pool with
`TaskPoolStrategy(size=n)`. A callable-class transform uses actors,
constructing the class once per worker, and defaults to an autoscaling actor
pool. Supply `ActorPoolStrategy(size=n)` when the actor count must be fixed.

```python
tasks = ds.map_batches(
    transform,
    compute=ray.data.TaskPoolStrategy(size=4),
    memory=1024 * 1024 * 1024,
)
actors = ds.map_batches(
    Predictor,
    compute=ray.data.ActorPoolStrategy(size=2),
)
```

Transformation arguments such as `memory`, `num_cpus`, and `num_gpus` are
logical resource requirements used by the scheduler. They do not impose
physical limits on the worker process.

## Row ordering

Transforms do not preserve block order by default. Ordering is preserved when
the Dataset is sorted or when it is enabled globally for the current data
context:

```python
ctx = ray.data.DataContext.get_current()
ctx.execution_options.preserve_order = True
```

This guarantee can reduce performance when workers finish unevenly, because
later blocks may need to wait for earlier ones.

## Placement groups per distributed replica

For a class-based transform that launches internal tasks or actors, pass
`ray_remote_args_fn`. The function can create a separate placement group and
scheduling strategy for each model replica. Enable child-task capture to
place the replica's internally launched work in the same group.

```python
from ray.util.scheduling_strategies import PlacementGroupSchedulingStrategy

def remote_args():
    pg = ray.util.placement_group([{"CPU": 1}, {"CPU": 1}])
    return {
        "scheduling_strategy": PlacementGroupSchedulingStrategy(
            placement_group=pg,
            placement_group_capture_child_tasks=True,
        )
    }

ds = ds.map_batches(DistributedModel, ray_remote_args_fn=remote_args)
```

Because `ray_remote_args_fn` is evaluated for replicas, do not create one
placement group outside the function when each replica needs isolated
bundles.

## Async transforms

An asynchronous transform must be a callable class whose `__call__` method is
`async def`. Function transforms are not supported for this mode. The feature
currently requires `uvloop==0.21.0`.

```python
class AsyncTransform:
    async def __call__(self, batch):
        return batch

ds = ds.map_batches(AsyncTransform)
```

## Column expressions

Column expressions are an Alpha API. Build them with `col()` and `lit()` and
apply them through `with_column()` so the optimizer can reason about and
reorder column operations.

Custom expression UDFs are vectorized over PyArrow arrays and must declare
their output with `return_dtype`.

```python
import pyarrow as pa
import pyarrow.compute as pc
from ray.data.datatype import DataType
from ray.data.expressions import col, udf

@udf(return_dtype=DataType.int32())
def add_one(values: pa.Array) -> pa.Array:
    return pc.add(values, 1)

ds = ds.with_column("value_plus_one", add_one(col("value")))
```
