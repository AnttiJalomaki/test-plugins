# Runtime, Adapters, and Packages

Use this reference before upgrading Python, Core, adapters, or project package
resolution. Relevant extraction sections: 1.9.0, 1.10-behavior-changes,
1.10.0, 1.11-udfs, 1.11.0, and 1.12.0.

## Python support

- The 1.9 line removes Python 3.8 support.
- Core 1.10 adds Python 3.13 support.
- Core 1.11 removes Python 3.9 support, making Python 3.10 the floor.
- Core 1.12 supports Python 3.14.

Check adapter Python support as well as Core before changing the project
runtime.

## Core and dependency constraints

The 1.9 line raises the minimum `dbt-adapters` version to 1.9.0.

Core 1.10 supports Pydantic v1 or v2. Its patch line also:

- raises the minimum JSON Schema package to 4.19.1;
- moves to Protobuf 6;
- caps `sqlparse` below 0.5.5;
- raises `dbt-common` to at least 1.37.3.

Adapter bounds change within that patch line. From 1.10.10 the
`dbt-adapters` range starts at 1.16.5. Version 1.10.21 temporarily caps it
below 1.24; 1.10.22 restores the upper bound to below 2.0.

Core 1.12 supports Python 3.14 and raises minimum dependencies to Click 8.3.0,
`dbt-common` 1.37.3, and `dbt-adapters` 1.24.5.

Resolve the full compatible set rather than upgrading one of Core, the
adapter, or `dbt-common` in isolation.

## Databricks materialization behavior

dbt-databricks 1.10.0 introduces `use_materialization_v2`, disabled by
default, to choose its restructured materializations:

```yaml
flags:
  use_materialization_v2: true
```

It uses project-level behavior-flag configuration. No maturity release is
specified, so inspect the installed adapter rather than assuming a later
default.

## Snowflake identifier quoting

From Core 1.10.11, project-level `quoting.snowflake_ignore_case` is a no-op.
Projects must not rely on it to change identifier casing.

Managed JavaScript functions have a separate Snowflake-specific argument
quoting config:

```yaml
config:
  snowflake:
    quote_args: true
```

This UDF setting does not restore the inert project-level relation behavior.

## Managed function adapter matrix

SQL managed functions work on BigQuery, Snowflake, Redshift, Postgres, and
Databricks. Python functions work on Snowflake, BigQuery, and Databricks with
Unity Catalog. JavaScript functions work on Snowflake and BigQuery from Core
1.12.

Body conventions and config support differ:

- BigQuery, Snowflake, and Databricks SQL bodies are expressions; Redshift and
  Postgres use `SELECT`.
- SQL argument defaults work only on Snowflake and Postgres.
- Snowflake and BigQuery Python require `runtime_version` and `entry_point`.
- Databricks accepts those two properties for compatibility but ignores them
  with a warning and embeds the body verbatim.
- BigQuery ignores SQL/Python `volatility` with a warning. For JavaScript it
  supports `deterministic` and `non-deterministic`, but not `stable`.

SQL overloads work on Snowflake and Postgres. Python and JavaScript overloads
work on Snowflake.

## Snapshot adapter support

The `hard_deletes` snapshot setting is supported by PostgreSQL, BigQuery,
Snowflake, and Redshift. Existing relations still require a manual schema and
data migration before switching deletion mode.

## Private Git packages

Core 1.12 supports private Git packages in both `packages.yml` and
`dependencies.yml`. dbt resolves package URLs from a configured environment
variable when one is present; otherwise it constructs an SSH URL. Ensure CI
has the corresponding environment value or SSH credentials instead of
assuming the resolution path is identical across environments.

## External parser dependency

Using Core 1.12's V2 parser delegation requires
`dbt-core-experimental-parser>=2.0.0a4`. The parser runs as an external command
and returns `manifest.json`, so its executable and environment are runtime
dependencies of every delegated parse.
