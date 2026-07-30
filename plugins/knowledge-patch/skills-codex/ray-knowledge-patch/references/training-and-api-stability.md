# Training and API stability

## Train V2 activation and migration

Starting in Ray 2.43, enable the overhauled Train V2 implementation and APIs
with `RAY_TRAIN_V2_ENABLED=1`.

```bash
RAY_TRAIN_V2_ENABLED=1 python train.py
```

The separate V1 reference is deprecated. It contains legacy framework
helpers, trainers, configuration paths, and session utilities. When migrating,
check the V2 API explicitly instead of assuming those old imports or paths
remain available.

## Dataset splitting across workers

By default, Train uses `Dataset.streaming_split()` to give every worker a
disjoint shard of every Dataset.

Use `DataConfig(datasets_to_split=[...])` to shard only named datasets:

```python
trainer = TorchTrainer(
    train_loop_per_worker,
    datasets={"train": train_ds, "val": val_ds},
    dataset_config=ray.train.DataConfig(datasets_to_split=["train"]),
    scaling_config=ScalingConfig(num_workers=2),
)
```

In this example, every worker can access the full `val` Dataset. If `val`
were split, validation results would need to be aggregated across workers.

## Labeled subclusters for Train data work

Pinning all data work requires two selectors because Train replaces a
Dataset's execution options during ingestion:

1. Copy the current `DataContext` and set `label_selector` while constructing
   the Dataset. This pins file listing and schema work.
2. Set the same selector under `DataConfig.execution_options` for the Dataset.
   This pins per-worker ingestion.

```python
from ray.data import ExecutionOptions
from ray.train import DataConfig

ctx = ray.data.DataContext.get_current().copy()
ctx.execution_options.label_selector = {"ray-subcluster": "data"}
with ray.data.DataContext.current(ctx):
    train_ds = ray.data.read_parquet(...)

trainer = TorchTrainer(
    ...,
    datasets={"train": train_ds},
    dataset_config=DataConfig(
        datasets_to_split=["train"],
        execution_options={
            "train": ExecutionOptions(
                label_selector={"ray-subcluster": "data"}
            )
        },
    ),
)
```

The construction-time selector alone does not pin per-worker ingest.

## Persisting a fitted preprocessor

Apply preprocessing before creating the Trainer. Serialize the fitted
preprocessor into the Trainer's `metadata`. Trainer metadata is available
through `TrainContext.get_metadata()` and is attached to checkpoints saved by
the Trainer.

Preprocessor serialization returns bytes, but Trainer metadata must be JSON
compatible. Base64-encode before constructing the Trainer and decode when
restoring from checkpoint metadata:

```python
payload = base64.b64encode(scaler.serialize()).decode("ascii")
trainer = TorchTrainer(..., metadata={"preprocessor_pkl": payload})
result = trainer.fit()

payload = result.checkpoint.get_metadata()["preprocessor_pkl"]
restored = StandardScaler.deserialize(base64.b64decode(payload))
```

## Per-Dataset object-store backpressure

Set an object-store limit through each Dataset's execution context:

```python
train_ds.context.execution_options.resource_limits = ray.data.ExecutionResources(
    object_store_memory=50 * 1024**3,
)
```

Ray Data slows production at the limit. This helps prevent training consumers
from being overrun by data that spills from the object store.

## API exposure and deprecation

An API with no annotation is a Developer API by default and can change.
Public and deprecated APIs require explicit annotations.

Stable API demotions and parameter changes require:

- warnings;
- a transition period in which old and new parameters coexist; and
- a deadline of six months or 25 minor versions.

Beta API changes use a deadline of three months or 12 minor versions.
Alpha APIs carry no stability guarantee.
