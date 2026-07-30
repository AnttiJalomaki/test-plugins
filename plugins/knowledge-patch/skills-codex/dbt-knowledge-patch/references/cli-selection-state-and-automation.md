# CLI, Selection, State, and Automation

Use this reference for command wrappers, selectors, state/defer workflows, and
CI behavior. Relevant extraction sections: 1.9.0, 1.10.0, 1.11.0, and 1.12.0.

## Quiet machine-readable output

`dbt show` and `dbt compile` retain their parseable JSON or text result when
run with `--quiet`. Automation can therefore suppress event logs without
losing the actual command result.

## Test and resource selection

The `unit_test:` selection method targets unit tests directly:

```bash
dbt test --select "unit_test:test_order_total"
```

`dbt test` accepts `--resource-type` and `--exclude-resource-type`; matching
environment-variable flags are also available.

Managed function resources can be listed or built with normal selectors:

```bash
dbt list --select "resource_type:function"
dbt build --select "resource_type:function"
dbt build --select is_positive_int
```

Core 1.11 allows nested paths in JSON output keys:

```bash
dbt ls --output json --output-keys name config.materialized
```

## Named-selector composition

A selector definition can call another named selector with the `selector`
method (since 1.12.0):

```yaml
selectors:
  - name: daily
    definition: {method: tag, value: daily}
  - name: daily_alias
    definition: {method: selector, value: daily}
```

## State selection and defer

With `--favor-state`, dbt favors a deferred relation only when its node is not
selected in the current command. Selection therefore still controls whether
the target environment is built.

`state_modified_compare_more_unrendered_values` makes `state:modified` compare
additional unrendered database, schema, and source properties while ignoring
rendered Jinja in configs:

```yaml
flags:
  state_modified_compare_more_unrendered_values: true
```

Managed `function()` calls participate in the DAG and in `state:modified` for
body, config, argument, and return-type changes. With `--defer` and a state
manifest, a function not selected or not yet built resolves to the deferred
environment's existing function.

## Hook failure behavior

Set `skip_nodes_if_on_run_start_fails` when a failed `on-run-start` hook should
skip all selected nodes:

```yaml
flags:
  skip_nodes_if_on_run_start_fails: true
```

## Ad-hoc SQL

Core 1.12 lets `dbt run-operation --sql` execute an ad-hoc SQL or Jinja
statement without a wrapper macro:

```bash
dbt run-operation --sql 'select count(*) from {{ ref("orders") }}'
```

Macros invoked through `run-operation` may call `ref()` on private and
protected models.

## Docs server binding

`dbt docs serve` accepts `--host` and defaults to `127.0.0.1`. Bind to all
interfaces only when remote access to the generated docs is intentional:

```bash
dbt docs serve --host 0.0.0.0
```

## Working directory guarantees

`dbt deps`, `dbt clean`, and `dbt init` no longer change the process working
directory. Embedded callers that relied on this side effect must choose and
manage paths explicitly.

## Exit status

From 1.9.1, a `PartialSuccess` result produces a nonzero exit status. CI and
wrappers must inspect it as a failure or partial outcome rather than assuming
the process exits successfully.

## Sample, empty, and retry flags

- `dbt build` supports `--sample`; sampling reaches referenced seeds and
  snapshot dependency graphs.
- Snapshots and seeds support `--empty` for schema-only execution.
- Microbatch backfills require both `--event-time-start` and
  `--event-time-end`.
- `dbt retry` honors `--threads`; for microbatch models it retries only failed
  batches and, from 1.10.20, retains the original invocation time.

## Parser command delegation

Core 1.12 can delegate parsing:

```bash
dbt parse --use-v2-parser \
  --v2-parser "dbt-core-experimental-parser parse"
```

The parser executable can instead come from `DBT_ENGINE_V2_PARSER` or project
flags. The integration loads the external `manifest.json` into the runtime.
