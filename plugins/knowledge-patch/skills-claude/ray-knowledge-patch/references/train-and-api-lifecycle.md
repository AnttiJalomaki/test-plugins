# Ray Train and API Lifecycle

## Train V2 opt-in and V1 migration

Starting in Ray 2.43, enable Train V2 with an environment variable before
starting the application:

```bash
RAY_TRAIN_V2_ENABLED=1 python train.py
```

The separate V1 reference is deprecated. Its legacy surface includes
framework helpers, trainers, configuration paths, and session utilities.
Migration should verify the V2 API and its imports rather than assume that V1
names and behavior carry over.

## Selecting Trainer datasets to split

By default, Train uses `Dataset.streaming_split()` to provide every worker a
disjoint shard of every Dataset. Set `DataConfig.datasets_to_split` when only
selected inputs should be sharded.

```python
trainer = TorchTrainer(
    train_loop_per_worker,
    datasets={"train": train_ds, "val": val_ds},
    dataset_config=ray.train.DataConfig(datasets_to_split=["train"]),
    scaling_config=ScalingConfig(num_workers=2),
)
```

In this example, every worker gets a disjoint training shard and a complete
copy of the unlisted validation Dataset. If validation is included in
`datasets_to_split`, aggregate its results across workers.

## Pinning data work to a labeled subcluster

Pin both phases of Dataset work:

1. Copy the current `DataContext`, set
   `execution_options.label_selector`, and make that context current while
   constructing the Dataset. This pins file listing and schema work.
2. Repeat the selector under the Dataset name in
   `DataConfig.execution_options`. This pins per-worker ingestion.

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

Train replaces the Dataset context's execution options, so the selector used
during construction does not by itself pin ingestion.

## Persisting a fitted preprocessor

Apply preprocessing before creating the Trainer. Serialize the fitted
preprocessor and store it in Trainer `metadata`. Since serialization returns
bytes and metadata must be JSON-compatible, base64-encode the payload.

```python
payload = base64.b64encode(scaler.serialize()).decode("ascii")
trainer = TorchTrainer(..., metadata={"preprocessor_pkl": payload})
result = trainer.fit()

payload = result.checkpoint.get_metadata()["preprocessor_pkl"]
restored = StandardScaler.deserialize(base64.b64decode(payload))
```

Trainer metadata is exposed through `TrainContext.get_metadata()` and is
attached to checkpoints saved by the Trainer, making the fitted state
available during restoration.

## Per-Dataset object-store backpressure

Set a Dataset's execution-context resource limit to slow production when that
Dataset reaches its allotted object-store memory. This helps keep training
consumers from being overrun by spilled data.

```python
train_ds.context.execution_options.resource_limits = ray.data.ExecutionResources(
    object_store_memory=50 * 1024**3,
)
```

The limit is per Dataset, so apply it to each producer that needs independent
backpressure.

## API exposure and deprecation windows

An unannotated API is a Developer API by default. Public and deprecated APIs
require explicit annotations.

For Stable APIs, demotion or parameter replacement requires:

- a warning;
- a transition where old and new parameters coexist; and
- a deadline of six months or 25 minor versions.

Beta transitions use three months or 12 minor versions. Alpha APIs carry no
stability guarantee.
