---
name: dbt-knowledge-patch
description: dbt Core
version: 1.12.0
license: MIT
metadata:
  author: Nevaberry
---


# dbt Core Knowledge Patch

Use this skill when changing dbt projects, command automation, resource YAML,
incremental pipelines, snapshots, tests, managed functions, or integrations
whose behavior depends on recent dbt Core releases.

Treat the project's installed Core and adapter versions, manifests, compiled
artifacts, and observed behavior as authoritative. Adapter support is not
uniform; check the relevant adapter note before adopting a Core feature.

## Reference index

| Reference | Topics |
| --- | --- |
| [Execution, incremental models, and freshness](references/execution-incremental-and-freshness.md) | Microbatch configuration, backfills, retries, sample and empty modes, source and model freshness |
| [Resources, tests, snapshots, and functions](references/resources-tests-and-functions.md) | Hard-delete snapshots, test configuration, constraints, managed SQL/Python/JavaScript UDFs, model pointers |
| [Configuration, validation, and parsing](references/configuration-validation-and-parsing.md) | Behavior flags, schema diagnostics, deprecated interfaces, catalogs, external parser, project inputs |
| [CLI, selection, state, and automation](references/cli-selection-state-and-automation.md) | Quiet output, selectors, state/defer behavior, docs serving, run-operation, process and exit behavior |
| [Semantic metadata and artifacts](references/semantic-metadata-and-artifacts.md) | Semantic Layer and OSI YAML, resource metadata, logs, manifests, compiled output |
| [Runtime, adapters, and packages](references/runtime-adapters-and-packages.md) | Python floors, dependency bounds, adapter-specific behavior, private Git packages |

## Breaking defaults and migrations

### Resource names and freshness hooks

Core 1.10 defaults these behavior flags to `true`:

```yaml
flags:
  require_resource_names_without_spaces: true
  source_freshness_run_project_hooks: true
```

Resource names containing spaces are rejected, and `dbt source freshness`
runs project hooks. A version-controlled `false` is only a temporary migration
opt-out and still warns; Core 2.0 removes both flags and always applies the new
behavior.

### Generic test arguments

`require_generic_test_arguments_property` arrived disabled in 1.10.5 and
defaults to `true` from 1.10.8. Put arguments beneath `arguments`:

```yaml
data_tests:
  - accepted_values:
      arguments:
        values: [placed, shipped, completed]
```

### Validation and warn-error behavior

Core 1.12 defaults both `validate_macro_args` and
`require_all_warnings_handled_by_warn_error` to `true`. Documented macro
arguments are checked against definitions, and unhandled warnings can fail a
command using `--warn-error`. JSON Schema deprecation warnings are also raised
by default.

Custom `generate_schema_name` macros must return a valid schema. Enable
`require_valid_schema_from_generate_schema_name` while migrating; a null return
is deprecated. Source and Semantic Model names with spaces warn, and
`REQUIRE_SOURCE_AND_SEMANTIC_MODEL_NAMES_WITHOUT_SPACES` promotes that check to
an error.

### Deprecated interfaces

Replace these interfaces before they disappear:

- Use `--select` instead of `--models`, `--model`, or `-m`.
- Stop using `dbt source freshness --output` or `-o`.
- Replace source `overrides` and Jinja `modules.itertools` usage.
- Replace `include`/`exclude` terminology in warn-error options.
- Do not depend on project-level `quoting.snowflake_ignore_case`; it is inert
  from 1.10.11.

## Microbatch incremental models

Use `microbatch` for large time-series relations. Model SQL describes one
batch and normally needs no `is_incremental()` filter:

```sql
{{ config(
    materialized='incremental',
    incremental_strategy='microbatch',
    event_time='event_occurred_at',
    begin='2020-01-01',
    batch_size='day',
    lookback=3
) }}

select * from {{ ref('stg_events') }}
```

`begin`, `event_time`, and `batch_size` are required. Batch size is `hour`,
`day`, `month`, or `year`; `lookback` defaults to one batch.
`concurrent_batches` overrides automatic parallelism detection.

Set `event_time` on every direct `ref` or `source` parent that should be
auto-filtered. An unconfigured parent is fully scanned for every batch; call
`.render()` on a configured relation to opt it out deliberately. PostgreSQL
also needs `unique_key`; Spark and BigQuery need `partition_by`.

Backfills require both UTC bounds:

```bash
dbt run --event-time-start "2024-09-01" --event-time-end "2024-09-04"
```

`dbt retry` reruns failed batches, honors `--threads`, and from 1.10.20 derives
batches from the original invocation time. Pre-hooks run only on the first
batch and post-hooks only on the last. Model Jinja can inspect `batch`.

Custom microbatch strategies require
`require_batched_execution_for_custom_microbatch_strategy` in project flags.

## Snapshot deletion semantics

Prefer `hard_deletes` over the legacy `invalidate_hard_deletes` setting:

```yaml
snapshots:
  - name: customer_snapshot
    config:
      unique_key: id
      strategy: timestamp
      updated_at: updated_at
      hard_deletes: new_record
```

`ignore` is the default. `invalidate` closes the current record by setting
`dbt_valid_to`; `new_record` appends a deletion record and adds
`dbt_is_deleted`. Never configure both old and new settings. Existing snapshot
tables are not migrated automatically, so migrate schema and data before
changing modes.

## Managed functions

Define managed UDF resources with a body in `functions/` and properties that
declare the name, arguments, return type, and config. dbt creates, updates, or
renames functions before dependent models. Reference them through
`function()` so qualification, DAG dependencies, state comparison, and defer
behavior remain correct:

```sql
select {{ function('is_positive_int') }}(value)
from {{ ref('input_values') }}
```

```bash
dbt build --select "resource_type:function"
dbt build --select is_positive_int
```

Support differs by adapter and language. Read the function reference before
choosing SQL, Python, JavaScript, or overloads. Unit tests do not create
functions implicitly; first build the tested model and ancestors with
`dbt build --select "+my_model_to_test" --empty`.

## Freshness, sampling, and schema-only runs

Source or table freshness can use `loaded_at_field` or a SQL
`loaded_at_query`. Model freshness is config-only. In 1.10, `build_after`
requires `count` and `period`; 1.11 adds an upstream-update-only form such as
`updates_on: any` without those fields.

Sample mode is available to `dbt build`; the final CLI expresses its window
through `--sample`, and sampling follows referenced seeds and snapshot
dependency graphs.

Use `dbt seed --empty` to create seed relations without rows. Snapshots also
accept `--empty`, and Jinja can branch on `flags.EMPTY` for schema-only work.

## Selection, state, and automation cautions

- `unit_test:` directly selects unit tests; `dbt test` also has
  `--resource-type` and `--exclude-resource-type` plus environment forms.
- Under `--favor-state`, defer wins only for a node not selected by the
  current command.
- `state_modified_compare_more_unrendered_values` compares more unrendered
  database, schema, and source properties while ignoring rendered config
  Jinja.
- `skip_nodes_if_on_run_start_fails` skips selected nodes after a failed
  `on-run-start` hook.
- A `PartialSuccess` result exits nonzero from 1.9.1; CI must not treat it as
  success.
- `dbt deps`, `dbt clean`, and `dbt init` no longer change the process working
  directory. Embedded callers must manage paths themselves.

## Adoption checklist

1. Inspect installed Core, adapter, Python, and dependency versions.
2. Search `dbt_project.yml` for behavior flags and temporary legacy opt-outs.
3. Run parse/schema diagnostics and expand summarized violations when needed.
4. Confirm resource YAML uses current property locations and unique names.
5. Verify adapter support before enabling UDFs, microbatch, or catalog modes.
6. Exercise selection, state/defer, retries, and exit status in CI.
7. Inspect generated manifests, structured logs, and compiled SQL where the
   integration consumes them.
