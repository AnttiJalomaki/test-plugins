---
name: dbt-knowledge-patch
description: dbt Core
version: 1.12.0
license: MIT
metadata:
  author: Nevaberry
---


# dbt Core Knowledge Patch

## When to load this skill

Load this skill when working with a dbt Core project, especially when:

- upgrading Core, an adapter, or Python;
- editing `dbt_project.yml`, resource properties, selectors, packages, or catalogs;
- implementing incremental models, snapshots, freshness, tests, or managed functions;
- automating the CLI, parsing artifacts, using state or defer, or embedding dbt;
- diagnosing behavior flags, validation warnings, parser changes, or deprecated syntax.

First inspect the project's pinned Core and adapter versions. Apply only guidance
available in that project version, and prefer its manifests, code, generated
artifacts, and test behavior when they disagree with general guidance.

## Reference index

| Reference | Topics |
| --- | --- |
| [behavior-flags-and-migrations.md](references/behavior-flags-and-migrations.md) | Default changes, opt-ins, validation, deprecated interfaces, migration hazards |
| [incremental-snapshots-and-freshness.md](references/incremental-snapshots-and-freshness.md) | Microbatch models, snapshots, sample and empty modes, source and model freshness, version pointers |
| [testing-selection-and-state.md](references/testing-selection-and-state.md) | Unit and data tests, selectors, state comparison, defer, constraints, retry |
| [project-config-and-semantic-layer.md](references/project-config-and-semantic-layer.md) | Resource metadata, Semantic Layer, catalogs, project inputs, packages, analyses |
| [managed-functions.md](references/managed-functions.md) | SQL, Python, and JavaScript UDF resources, overloads, references, adapter support |
| [cli-artifacts-and-runtime.md](references/cli-artifacts-and-runtime.md) | CLI automation, docs server, parsers, artifacts, logs, working directory, dependencies |

## Breaking changes and deprecations

### Put behavior flags in version control

Use the top-level `flags` block in `dbt_project.yml` for compatibility decisions.
Do not rely on local CLI flags to preserve shared project semantics.

```yaml
flags:
  require_resource_names_without_spaces: true
  source_freshness_run_project_hooks: true
  require_generic_test_arguments_property: true
  validate_macro_args: true
  require_all_warnings_handled_by_warn_error: true
```

Important defaults and migrations:

- Resource names containing spaces are rejected, and `dbt source freshness`
  runs project hooks. Setting either behavior to `false` is temporary and warns.
- Generic-test inputs belong under `arguments`.
- Macro argument documentation is checked against definitions; warning handling
  can fail commands that use `--warn-error`.
- JSON Schema deprecation diagnostics are raised by default. Duplicate YAML
  keys, unsupported properties, malformed resource files, and invalid inline
  `config()` calls can now surface during parsing.
- A custom `generate_schema_name` macro must return a valid schema rather than
  null when its validation flag is enabled.

See [behavior-flags-and-migrations.md](references/behavior-flags-and-migrations.md)
for maturity timing, adapter-specific opt-ins, and the complete deprecated list.

### Replace deprecated interfaces

Prefer current interfaces in all new automation:

| Replace | With |
| --- | --- |
| `dbt run --models`, `--model`, or `-m` | `dbt run --select` |
| `dbt source freshness --output` or `-o` | Consume the supported result/artifact path |
| source `overrides` | Current source property/config structure |
| `modules.itertools` in Jinja | Project or package logic that does not use it |
| direct generic-test keys | An `arguments:` mapping |
| `quoting.snowflake_ignore_case` | Explicit identifier conventions and quoting |

`quoting.snowflake_ignore_case` is inert; leaving it in a project does not alter
identifier casing. A `PartialSuccess` result also exits nonzero, so CI must use
the command status rather than assuming partial success is a zero exit.

### Respect runtime floors

- Do not run modern projects on Python 3.8 or 3.9.
- Pin Core, adapters, and companion packages as a compatible set.
- When upgrading, verify Click, JSON Schema, Protobuf, Pydantic, `sqlparse`,
  `dbt-common`, and `dbt-adapters` constraints before debugging application SQL.

## Microbatch quick reference

Use `microbatch` for large time-series incremental models whose work can be
split into replaceable time windows.

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

Key rules:

- `begin`, `event_time`, and `batch_size` are required; sizes are `hour`, `day`,
  `month`, and `year`.
- dbt filters direct `ref()` and `source()` parents only when each parent has
  `event_time`; otherwise that parent is fully scanned for every batch.
- Call `.render()` on a configured parent relation to opt it out of filtering.
- `lookback` defaults to one batch. `concurrent_batches` can override detected
  parallelism.
- PostgreSQL also needs `unique_key`; Spark and BigQuery need `partition_by`.
- Backfills require both `--event-time-start` and `--event-time-end`; all event
  times and bounds are UTC.
- Hooks bracket the full microbatch run: pre-hooks on the first batch and
  post-hooks on the last. Retry reruns failed batches only.

Full setup and retry semantics are in
[incremental-snapshots-and-freshness.md](references/incremental-snapshots-and-freshness.md).

## Snapshot and freshness quick reference

Choose snapshot hard-delete behavior explicitly:

```yaml
snapshots:
  - name: customers_snapshot
    config:
      unique_key: id
      strategy: timestamp
      updated_at: updated_at
      hard_deletes: new_record
```

- `ignore` is the default.
- `invalidate` closes the current record by setting `dbt_valid_to`.
- `new_record` adds a deletion row and the `dbt_is_deleted` column.
- Do not combine `hard_deletes` with `invalidate_hard_deletes`.
- Existing snapshot relations are not migrated automatically; migrate their
  schema and data before changing modes.

Freshness can be query-driven:

```yaml
sources:
  - name: raw
    tables:
      - name: events
        config:
          loaded_at_query: "select max(_loaded_at) from raw.events"
```

Model freshness is config-only. `build_after` accepts a `count`/`period` pair or
an update trigger such as `updates_on: any`.

## Managed-function quick reference

Define a function body under `functions/` and its signature in a properties
file. Use `function()` so dbt records the dependency and qualifies the name.

```sql
select {{ function('is_positive_int') }}(value)
from {{ ref('input_values') }}
```

```yaml
functions:
  - name: is_positive_int
    arguments:
      - {name: a_string, data_type: string}
    returns: {data_type: integer}
```

Function bodies, adapters, and language support differ substantially. Read
[managed-functions.md](references/managed-functions.md) before selecting a file
extension, runtime, volatility, overload, or package configuration.

## Testing, state, and selection quick reference

- Select a unit test directly with `unit_test:<name>`.
- Use `--resource-type` and `--exclude-resource-type` to restrict `dbt test`.
- The legacy `tests:` property remains accepted alongside `data_tests:`.
- `--favor-state` favors a deferred relation only when its node is not selected.
- State comparison can include more unrendered relation properties while
  ignoring rendered Jinja when its behavior flag is enabled.
- Named selectors can compose other named selectors with `method: selector`.
- Function body, config, arguments, and return changes all affect
  `state:modified`.

See [testing-selection-and-state.md](references/testing-selection-and-state.md)
for flags, custom test configs, graph behavior, and package-ref resolution.

## CLI and project configuration quick reference

- `dbt show --quiet` and `dbt compile --quiet` retain their parseable command
  result while suppressing event logs.
- `dbt docs serve` binds to `127.0.0.1` by default; use `--host 0.0.0.0` only
  when remote access is intended and secured.
- `dbt deps`, `dbt clean`, and `dbt init` do not change the caller's working
  directory. Embedded callers must manage paths themselves.
- `dbt seed --empty` creates seed relations without rows.
- `dbt run-operation --sql` executes ad-hoc SQL or Jinja without a wrapper macro.
- `dbt ls --output json --output-keys` accepts nested paths such as
  `config.materialized`.
- `vars.yml`, `.env`, and Jinja-suffixed `.sql.jinja`/`.md.jinja` files are
  project inputs; handle secrets in `.env` with the same care as other secrets.

For artifact fields, external parsing, catalogs, package URL resolution, and
dependency details, use the indexed references rather than assuming an older
manifest or CLI shape.
