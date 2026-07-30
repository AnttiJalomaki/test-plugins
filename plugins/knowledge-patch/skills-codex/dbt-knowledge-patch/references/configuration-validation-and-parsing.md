# Configuration, Validation, and Parsing

Use this reference when migrating project flags, responding to schema
diagnostics, configuring catalogs, or replacing Core's parser. Relevant
extraction sections: 1.9.0, 1.10-behavior-changes, 1.10.0, 1.11.0, and
1.12.0.

## Matured behavior defaults

Core 1.10 changes these defaults from `false` to `true`:

```yaml
flags:
  require_resource_names_without_spaces: true
  source_freshness_run_project_hooks: true
```

Resource names containing spaces are rejected by default, and `dbt source
freshness` runs project hooks by default. Keep a temporary compatibility
opt-out in the version-controlled `flags` block of `dbt_project.yml`:

```yaml
flags:
  require_resource_names_without_spaces: false
  source_freshness_run_project_hooks: false
```

An explicit `false` keeps legacy behavior but continues to emit deprecation
warnings. Core 2.0 removes both flags and always applies the new behavior.

## Generic test arguments

Core 1.10.5 introduces `require_generic_test_arguments_property` disabled by
default; 1.10.8 changes the default to `true`. With the flag enabled, put all
generic-test parameters below `arguments`, not directly below the test name.

```yaml
flags:
  require_generic_test_arguments_property: true

models:
  - name: orders
    columns:
      - name: status
        data_tests:
          - accepted_values:
              arguments:
                values: [placed, shipped, completed]
```

## Macro and warning validation

Core 1.10 adds two disabled-by-default flags:

```yaml
flags:
  validate_macro_args: true
  require_all_warnings_handled_by_warn_error: true
```

`validate_macro_args` warns when documented macro arguments do not match the
macro definition; `--warn-error` promotes those warnings to errors.
`require_all_warnings_handled_by_warn_error` allows unhandled warnings to stop
a build when `--warn-error` is active. Both flags mature to default `true` in
Core 1.12.

## Resource and ref lookup flags

Core 1.11 adds:

```yaml
flags:
  require_unique_project_resource_names: true
  require_ref_searches_node_package_before_root: true
```

The first restores an error for duplicate node names inside one project. The
second changes an ambiguous package-internal `ref()` to search the referencing
node's package before the root project.

## Schema-name and resource-name validation

Core 1.12 raises JSON Schema-based deprecation warnings by default. A custom
`generate_schema_name` macro returning null is deprecated behind:

```yaml
flags:
  require_valid_schema_from_generate_schema_name: true
```

Source and Semantic Model names containing spaces warn.
`REQUIRE_SOURCE_AND_SEMANTIC_MODEL_NAMES_WITHOUT_SPACES` can promote this
validation to an error.

## Project and resource schema diagnostics

Core 1.10 begins JSON Schema validation for `dbt_project.yml` and resource
YAML. It also:

- detects duplicate YAML keys;
- validates `{{ config(...) }}` in model SQL even if static parsing is
  unavailable;
- warns about unexpected Jinja blocks;
- warns about unsupported custom keys or properties.

Some schema checks are adapter-gated and some begin as preview deprecations.
Diagnostics expose event names, summarize repeated violations, and can be
expanded to show every instance. Do not assume a summarized count identifies
all source locations.

## Deprecated CLI, YAML, and Jinja surfaces

Core 1.10 deprecates:

- `dbt source freshness --output` and `-o`;
- source `overrides`;
- `modules.itertools` in Jinja;
- `--models`, `--model`, and `-m` selection aliases in favor of `--select`;
- `include` and `exclude` terminology in warn-error options.

Use, for example:

```bash
dbt run --select my_model
```

## Catalog configuration

Core 1.10 parses `catalogs.yml`; from 1.10.12 it also parses that file during
`parse`, `seed`, and `test`. Catalog integration config accepts `file_format`.

Core 1.12 introduces opt-in Catalog configuration V2:

```yaml
flags:
  use_catalogs_v2: true
```

Catalog configuration is then loaded by every command that needs a manifest.
For V1 integrations, `catalog_database` can override the database for any
catalog type and has highest priority in database-name generation.

## External V2 parser

`--use-v2-parser` bypasses Core's parser, invokes an external parser, and loads
the resulting `manifest.json` back into the runtime manifest. Choose the
command through `--v2-parser`, `DBT_ENGINE_V2_PARSER`, or project flags. The
default command is `dbt-core-experimental-parser parse`.

```bash
dbt parse --use-v2-parser \
  --v2-parser "dbt-core-experimental-parser parse"
```

This integration requires `dbt-core-experimental-parser>=2.0.0a4`.

## Additional project inputs

Core 1.12 accepts project variables from `vars.yml`, loads environment
variables automatically from `.env`, and recognizes Jinja-suffixed SQL and
Markdown extensions such as `.sql.jinja` and `.md.jinja`.

## Test config opt-ins

Data tests can use `sql_header` behind
`require_sql_header_in_test_configs`. Unit tests and generic data tests can
pass custom `ref()` keyword arguments behind `support_custom_ref_kwargs`.

```yaml
flags:
  require_sql_header_in_test_configs: true
  support_custom_ref_kwargs: true
```

## Databricks materialization V2

dbt-databricks 1.10.0 adds project-level `use_materialization_v2`, disabled by
default, to opt into restructured materializations:

```yaml
flags:
  use_materialization_v2: true
```

No maturity release is specified, so do not assume the default has changed
without checking the installed adapter.
