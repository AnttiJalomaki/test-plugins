# Managed Functions

## Resource layout and lifecycle

The `1.11-udfs` batch introduces managed warehouse functions as DAG resources.
Put the body in `functions/<name>.sql` or `.py` and define the function's name,
return type, arguments, and config in a corresponding properties file. dbt
combines them into `CREATE FUNCTION` and creates, updates, or renames the
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
languages are unsupported.

## SQL adapter behavior

SQL functions are supported on BigQuery, Snowflake, Redshift, PostgreSQL, and
Databricks. BigQuery, Snowflake, and Databricks bodies are expressions;
Redshift and PostgreSQL bodies use a `SELECT`.

Argument defaults are available only on Snowflake and PostgreSQL. BigQuery
ignores `volatility` on SQL and Python functions with a warning; Snowflake
applies it.

## Python functions

Python function resources are supported on Snowflake, BigQuery, and Databricks
with Unity Catalog. Snowflake and BigQuery require `runtime_version` and
`entry_point`; both can install optional, version-pinned `packages` in the
warehouse.

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

Snowflake supports Python 3.10 through 3.13; BigQuery supports 3.11.

Databricks accepts `runtime_version` and `entry_point` for cross-adapter
compatibility but warns that they have no effect. It embeds the `.py` file
verbatim as the function body, so the file needs a top-level return rather than
being a standalone Python module:

```python
import re
def main(a_string):
    return 1 if re.search(r'^[0-9]+$', a_string or '') else 0
return main(a_string)
```

## JavaScript functions

Core 1.12 adds `.js` function bodies on Snowflake and BigQuery. JavaScript on
another adapter is a parse error.

```javascript
// functions/is_positive_int.js
return /^[0-9]+$/.test(a_string) ? 1 : 0;
```

Snowflake can quote argument names with `config.snowflake.quote_args`:

```yaml
config:
  snowflake:
    quote_args: true
```

BigQuery applies `deterministic` and `non-deterministic` volatility but does not
support `stable`.

## Overloads

Core 1.12 adds `overloads`, which assigns several argument signatures to one
function name. Each overload selects a separate body with `defined_in` and may
override `arguments` and `returns`. If `returns` is omitted, it inherits the
root return type.

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

SQL overloads are supported on Snowflake and PostgreSQL. Python and JavaScript
overloads are supported on Snowflake. All signatures share one DAG node and
are selected and built together; `dbt retry` reruns only failed overloads.

## References, state, and defer

Use `function()` rather than a hard-coded qualified warehouse name. It compiles
to the qualified function and creates a function-to-model DAG dependency:

```sql
select {{ function('is_positive_int') }}(value) as is_positive
from {{ ref('input_values') }}
```

Function body, config, argument, and return-type changes all affect
`state:modified`. With `--defer` and a state manifest, `function()` resolves to
the deferred environment's existing function if the function is not selected
or has not yet been built in the target.

Selection examples:

```bash
dbt list --select "resource_type:function"
dbt build --select "resource_type:function"
dbt build --select is_positive_int
```

## Unit tests

Unit tests do not create a warehouse function implicitly. Build it and the
tested model's ancestors first:

```bash
dbt build --select "+my_model_to_test" --empty
```
