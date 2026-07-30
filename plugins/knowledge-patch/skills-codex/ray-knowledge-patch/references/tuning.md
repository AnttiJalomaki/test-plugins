# Ray Tune

## Emitting function-trainable results

A function trainable has three result-emission forms:

- call `tune.report()` for intermediate metrics;
- return a dictionary for only the final result; or
- yield dictionaries for successive results.

```python
def objective(config):
    for score in calculate_scores(config):
        yield {"score": score}
```

Do not use `tune.report()` inside a class-based `Trainable`.

## Open-ended sampling with a time budget

Set `num_samples=-1` together with `time_budget_s` to keep creating trials
until the wall-clock budget expires.

```python
tuner = tune.Tuner(
    objective,
    tune_config=tune.TuneConfig(num_samples=-1, time_budget_s=3600),
)
```

A finite `num_samples` instead places a hard cap on the number of trials.

## Scheduler compatibility

Checkpoint requirements and search-algorithm support vary by scheduler:

| Scheduler | Checkpointing | Search algorithm compatibility |
| --- | --- | --- |
| ASHA | Requires no checkpointing | Compatible |
| Median Stopping | Requires no checkpointing | Compatible |
| HyperBand | Required | Compatible |
| BOHB | Required | Only `TuneBOHB` |
| PBT | Required | Incompatible |
| PB2 | Required | Incompatible |

Choose the search algorithm and checkpoint behavior together with the
scheduler rather than configuring them independently.

## Dynamic trial resources

`ResourceChangingScheduler` can wrap any other scheduler. Use it to change a
trial's resource requirements while tuning is in progress.
