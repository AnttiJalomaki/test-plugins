# Resources, Tests, Snapshots, and Functions

Use this reference for resource properties, unit and data tests, snapshot
deletion semantics, managed UDFs, and versioned-model pointers. Relevant
extraction sections: 1.9-guides, 1.9.0, 1.11-udfs, 1.11.0, and 1.12.0.

## Snapshot hard deletes

Snapshots support `hard_deletes` with three modes:

- `ignore` is the default and does not record a source deletion.
- `invalidate` closes the existing row by setting `dbt_valid_to`.
- `new_record` writes a deletion record and adds `dbt_is_deleted`.

```yaml
snapshots:
  - name: my_snapshot
    config:
      unique_key: id
      strategy: timestamp
      updated_at: updated_at
      hard_deletes: new_record
```

The legacy `invalidate_hard_deletes` setting remains accepted, but it cannot
be combined with `hard_deletes`. dbt does not migrate existing snapshot tables
automatically. Migrate the table schema and existing data before switching
modes, or use the new setting only for new snapshots. PostgreSQL, BigQuery,
Snowflake, and Redshift adapters support this config.

## Unit and data test behavior

Select unit tests directly with the `unit_test:` method:

```bash
dbt test --select "unit_test:test_order_total"
```

`dbt test` also accepts `--resource-type` and `--exclude-resource-type`, with
corresponding environment-variable flags. Unit tests can be disabled by config
and can use versioned refs.

The older `tests:` property remains accepted without a deprecation warning
alongside `data_tests:`. Data tests accept arbitrary config options; adapters
receive them through `pre_model` and `post_model` hooks.

Core 1.12 provides two test opt-ins:

```yaml
flags:
  require_sql_header_in_test_configs: true
  support_custom_ref_kwargs: true
```

The first permits `sql_header` in data-test configuration. The second permits
custom `ref()` keyword arguments from unit tests and generic data tests.

## Generic test argument layout

`require_generic_test_arguments_property` appeared disabled in 1.10.5 and
defaults to `true` from 1.10.8. Nest test parameters under `arguments`:

```yaml
models:
  - name: orders
    columns:
      - name: status
        data_tests:
          - accepted_values:
              arguments:
                values: [placed, shipped, completed]
```

## Column config and constraints

Columns may contain a `config` mapping. Column meta and tags propagate to the
tests defined on that column.

```yaml
models:
  - name: orders
    columns:
      - name: id
        config:
          meta:
            owner: analytics
```

Foreign-key constraint expressions can use `ref()` and `source()` instead of
hard-coded relation names:

```yaml
models:
  - name: orders
    columns:
      - name: customer_id
        constraints:
          - type: foreign_key
            expression: "{{ ref('customers') }} (id)"
```

## Managed function files

Core 1.11 manages scalar and aggregate warehouse functions as DAG resources.
Place each body in `functions/<name>.sql` or `.py` and define its name, return
type, arguments, and config in a corresponding properties file. dbt combines
the files into `CREATE FUNCTION`, then creates, updates, or renames the
function before dependent models.

```sql
-- functions/is_positive_int.sql (Snowflake expression body)
REGEXP_INSTR(a_string, '^[0-9]+$')
```

```yaml
functions:
  - name: is_positive_int
    config:
      schema: udf_schema
      volatility: deterministic
    arguments:
      - name: a_string
        data_type: string
    returns:
      data_type: integer
```

Only scalar and aggregate functions are supported. Java, Scala, and other UDF
languages are not supported.

## SQL function adapter rules

SQL functions are supported on BigQuery, Snowflake, Redshift, Postgres, and
Databricks. BigQuery, Snowflake, and Databricks bodies are expressions;
Redshift and Postgres bodies use a `SELECT`. Argument defaults are available
only on Snowflake and Postgres.

BigQuery ignores `volatility` on SQL and Python functions with a warning.
Snowflake applies it.

## Python function adapter rules

Python functions are supported on Snowflake, BigQuery, and Databricks with
Unity Catalog. Snowflake and BigQuery require `runtime_version` and
`entry_point`; both can install optionally version-pinned warehouse packages.
Snowflake supports Python 3.10–3.13, and BigQuery supports 3.11.

```yaml
functions:
  - name: is_positive_int
    config:
      runtime_version: "3.11"
      entry_point: main
      packages: [numpy, "pandas==1.5.0"]
    arguments:
      - {name: a_string, data_type: string}
    returns: {data_type: integer}
```

Databricks accepts `runtime_version` and `entry_point` only for cross-adapter
compatibility and warns that they have no effect. It embeds the `.py` file
verbatim as the function body, so the file needs a top-level return rather
than the shape of a standalone Python module:

```python
import re
def main(a_string):
    return 1 if re.search(r'^[0-9]+$', a_string or '') else 0
return main(a_string)
```

## JavaScript functions

Core 1.12 accepts `.js` bodies on Snowflake and BigQuery. JavaScript on another
adapter is a parse error.

```javascript
return /^[0-9]+$/.test(a_string) ? 1 : 0;
```

Snowflake can quote argument names through nested adapter config:

```yaml
config:
  snowflake:
    quote_args: true
```

BigQuery applies `deterministic` and `non-deterministic` volatility, but does
not support `stable`.

## Overloaded functions

Core 1.12 adds `overloads`, giving one function name multiple signatures. Each
overload uses `defined_in` to name a separate body and may override arguments
and returns. Omitting an overload return type inherits the root return type.

```yaml
functions:
  - name: is_positive_int
    arguments:
      - {name: a_string, data_type: string}
    returns: {data_type: integer}
    overloads:
      - defined_in: is_positive_int_numeric
        arguments:
          - {name: a_num, data_type: numeric}
```

SQL overloads work on Snowflake and Postgres. Python and JavaScript overloads
work on Snowflake. All signatures share one DAG node and are selected and
built together; `dbt retry` reruns only overloads that failed.

## Function references, selection, and state

Use `function()` instead of hard-coding a qualified warehouse name:

```sql
select {{ function('is_positive_int') }}(value) as is_positive
from {{ ref('input_values') }}
```

It compiles to the qualified function and records a function-to-model DAG
edge. Body, config, argument, and return-type changes all participate in
`state:modified`.

```bash
dbt list --select "resource_type:function"
dbt build --select "resource_type:function"
dbt build --select is_positive_int
```

With `--defer` and a state manifest, `function()` resolves to the deferred
environment's existing function when the function is not selected or has not
yet been built in the target environment.

Unit tests do not create functions implicitly. Build the function and tested
model's ancestors first:

```bash
dbt build --select "+my_model_to_test" --empty
```

## Versioned-model latest pointers

Core 1.12 can create an unversioned relation such as `dim_customers` pointing
to the latest version. Enable it project-wide or per model:

```yaml
flags:
  latest_version_pointer_enabled_by_default: true
```

The per-model property is `latest_version_pointer`. Pointer collision checks
respect identifier quoting and case. Unquoted floating versions such as
`v: 4.5` are no longer silently dropped.

## Model config metadata access

Model Jinja has `config.meta_get(key)` for optional metadata and
`config.meta_require(key)` for required metadata:

```jinja
{{ config(meta={"owner": "finance", "policy": "restricted"}) }}
{% set owner = config.meta_get("owner") %}
{% set policy = config.meta_require("policy") %}
```
