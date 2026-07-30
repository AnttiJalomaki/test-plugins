# Behavior Flags and Migrations

## Default behavior changes

The following behavior changes are part of the `1.10-behavior-changes` batch.

Core 1.10 changes `require_resource_names_without_spaces` and
`source_freshness_run_project_hooks` from `false` to `true`. Resource names with
spaces are rejected by default, and `dbt source freshness` runs project hooks by
default. A version-controlled compatibility opt-out is possible:

```yaml
flags:
  require_resource_names_without_spaces: false
  source_freshness_run_project_hooks: false
```

An explicit `false` retains legacy behavior but emits deprecation warnings.
Core 2.0 removes both flags and always applies the new behavior.

Core 1.10.5 introduces `require_generic_test_arguments_property` with a default
of `false`; in 1.10.8 it matures to `true`. When enabled, put generic-test
inputs under `arguments`:

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

Core 1.10 introduces two other opt-ins, both initially `false` and both default
`true` in Core 1.12:

```yaml
flags:
  validate_macro_args: true
  require_all_warnings_handled_by_warn_error: true
```

`validate_macro_args` warns when documented macro arguments do not match the
macro definition. Those warnings are errors under `--warn-error`.
`require_all_warnings_handled_by_warn_error` can stop a build when
`--warn-error` is set.

dbt-databricks 1.10.0 separately introduces `use_materialization_v2`, default
`false`, to select its restructured materializations. It uses the same
project-level `flags` block. No maturity release is specified.

## Validation and diagnostics

Core 1.10.0 begins JSON Schema validation of `dbt_project.yml` and resource
YAML. Validation also:

- detects duplicate YAML keys;
- validates `{{ config(...) }}` in model SQL even when static parsing is
  unavailable;
- warns about unexpected Jinja blocks and unsupported custom keys or
  properties;
- reports named diagnostic events, summarizes repeated violations, and can be
  expanded to display every instance.

Some schema checks are gated by adapter support, and some diagnostics begin as
preview deprecations.

In Core 1.12.0, JSON Schema deprecation warnings are raised by default. A custom
`generate_schema_name` macro that returns null is deprecated behind
`require_valid_schema_from_generate_schema_name`:

```yaml
flags:
  require_valid_schema_from_generate_schema_name: true
```

Source and Semantic Model names containing spaces warn. The
`REQUIRE_SOURCE_AND_SEMANTIC_MODEL_NAMES_WITHOUT_SPACES` setting can promote
that validation to an error.

## Deprecated and inert interfaces

Core 1.10 deprecates:

- `dbt source freshness --output` and `-o`;
- source `overrides`;
- `modules.itertools` in the Jinja context;
- `--models`, `--model`, and `-m` selection aliases; use `--select`;
- `include` and `exclude` terminology in warn-error options.

```bash
dbt run --select my_model
```

From Core 1.10.11, project-level `quoting.snowflake_ignore_case` is a no-op.
Projects must use deliberate identifier naming and quoting rather than relying
on this setting to change casing.

## Exit and runtime migration checks

Starting in 1.9.1 (from the `1.9.0` batch), `PartialSuccess` yields a nonzero
exit status. CI and wrappers must handle it as a failing process status even if
some nodes succeeded.

The 1.9 line removes Python 3.8 support. Core 1.11.0 removes Python 3.9 support,
so use Python 3.10 or newer there. Runtime package details are collected in
[cli-artifacts-and-runtime.md](cli-artifacts-and-runtime.md).
