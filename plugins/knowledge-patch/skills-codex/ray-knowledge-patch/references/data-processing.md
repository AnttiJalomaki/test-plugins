# Ray Data processing

## Lazy execution and streaming barriers

Dataset transformations are lazy. Once the Dataset is consumed,
non-shuffle operators can run concurrently as a streaming pipeline.

`sort()` and `groupby()` are barriers: they must materialize data, and
streaming stops until the shuffle finishes.

## Batch sizing

`map_batches()` accepts `batch_size="auto"` for automatic batch sizing.
A GPU transform instead requires an explicit integer batch size. Reduce that
integer if the worker runs out of memory.

```python
ds = ds.map_batches(
    GpuPredictor,
    num_gpus=1,
    batch_size=64,
)
```

## Tasks, actor pools, and logical resources

The transform form selects its execution strategy:

- A function uses tasks. Limit the task pool with
  `TaskPoolStrategy(size=n)`.
- A callable class uses actors. Its `__init__` runs once per worker, and it
  defaults to an autoscaling actor pool.
- Pass a fixed `ActorPoolStrategy` when a fixed class-worker pool is needed.

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

Transform arguments such as `memory`, `num_cpus`, and `num_gpus` are logical
scheduling requirements. They do not enforce physical memory, CPU, or GPU
limits.

## Row order

Transformations do not preserve block order by default. Ordering is preserved
when the Dataset is sorted or when `preserve_order` is enabled globally in the
current Data context.

```python
ctx = ray.data.DataContext().get_current()
ctx.execution_options.preserve_order = True
```

Enabling order preservation can reduce performance when workers finish at
different times. Enable it only when downstream behavior depends on row
order.

## Per-replica placement groups

For a class-based distributed model transform, pass `ray_remote_args_fn`.
The function can create a separate placement group and scheduling strategy
for every replica. Enable child-task capture to place actors or tasks launched
inside that replica into the same group.

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

## Async transforms

An asynchronous transform must be a callable class with
`async def __call__`. Async function transforms are not supported. This
feature requires `uvloop==0.21.0`.

```python
class AsyncTransform:
    async def __call__(self, batch):
        return batch

ds = ds.map_batches(AsyncTransform)
```

## Column expressions

Column expressions are an alpha API. Build expressions with `col()` and
`lit()` and apply them through `with_column()`. This representation lets the
optimizer reason about and reorder column operations.

Custom expression UDFs are vectorized over PyArrow arrays and must declare a
`return_dtype`.

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
