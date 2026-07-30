# Ray Tune Scheduling

## Function-trainable result emission

A function trainable has three result-emission patterns:

- call `tune.report()` for intermediate metrics;
- return a dictionary for only the final result; or
- yield dictionaries for successive results.

```python
def objective(config):
    for score in calculate_scores(config):
        yield {"score": score}
```

Do not call `tune.report()` from a class-based `Trainable`; it is supported
for function trainables.

## Time-budgeted open-ended sampling

Set `num_samples=-1` with `time_budget_s` to keep generating trials until the
wall-clock budget expires.

```python
tuner = tune.Tuner(
    objective,
    tune_config=tune.TuneConfig(num_samples=-1, time_budget_s=3600),
)
```

If `num_samples` is finite, it caps the number of trials even when a time
budget is also present.

## Scheduler compatibility

Checkpoint requirements and search-algorithm compatibility vary by scheduler:

| Scheduler | Checkpointing | Search algorithm |
| --- | --- | --- |
| ASHA | Must be disabled | Compatible |
| Median Stopping | Must be disabled | Compatible |
| HyperBand | Required | Compatible |
| BOHB | Required | Only compatible with `TuneBOHB` |
| PBT | Required | Incompatible |
| PB2 | Required | Incompatible |

Choose the search algorithm and checkpoint policy together with the scheduler;
they are not independent options.

## Changing trial resources

`ResourceChangingScheduler` can wrap any other scheduler and adjust trial
resource requirements while tuning is in progress. Put the underlying
scheduler inside this wrapper when resource changes need to coexist with its
stopping or promotion policy.
