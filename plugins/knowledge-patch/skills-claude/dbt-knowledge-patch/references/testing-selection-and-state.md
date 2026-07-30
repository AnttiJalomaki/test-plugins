# Testing, Selection, and State

## Unit and data tests

Core 1.9.0 adds the `unit_test:` selection method and lets `dbt test` accept
`--resource-type`, `--exclude-resource-type`, and their environment-variable
counterparts.

```bash
dbt test --select "unit_test:test_order_total"
```

Data tests accept arbitrary configuration that is passed to adapter
`pre_model` and `post_model` hooks. Unit tests can be disabled through config
and can use versioned refs. The older `tests:` key remains accepted without a
deprecation warning alongside `data_tests:`.

Generic-test inputs move under `arguments` when
`require_generic_test_arguments_property` is enabled; see
[behavior-flags-and-migrations.md](behavior-flags-and-migrations.md).

Core 1.12.0 adds these test opt-ins:

```yaml
flags:
  require_sql_header_in_test_configs: true
  support_custom_ref_kwargs: true
```

`require_sql_header_in_test_configs` allows data tests to use `sql_header`.
`support_custom_ref_kwargs` allows unit tests and generic data tests to pass
custom keyword arguments to `ref()`.

The Jinja `graph` includes unit tests in Core 1.12.0.

## Foreign-key constraints

Foreign-key expressions can use `ref()` and `source()` instead of hard-coded
relation names (from `1.9.0`):

```yaml
models:
  - name: orders
    columns:
      - name: customer_id
        constraints:
          - type: foreign_key
            expression: "{{ ref('customers') }} (id)"
```

## State selection and failure behavior

With `--favor-state`, a deferred relation is favored only if its node is not
selected in the current command. Two Core 1.9.0 flags refine state comparison
and hook failure:

```yaml
flags:
  state_modified_compare_more_unrendered_values: true
  skip_nodes_if_on_run_start_fails: true
```

`state_modified_compare_more_unrendered_values` makes `state:modified` compare
additional unrendered database, schema, and source properties while ignoring
rendered Jinja in configs. `skip_nodes_if_on_run_start_fails` turns selected
nodes into skipped nodes after a failed `on-run-start` hook.

For managed functions (from `1.11-udfs`), changes to the body, config,
arguments, or return type are all detected by `state:modified`. With `--defer`
and a state manifest, `function()` uses the deferred environment's existing
function when the node is not selected or has not yet been built in the target.

## Named-selector composition

Core 1.12.0 selector definitions can reference another named selector with the
`selector` method:

```yaml
selectors:
  - name: daily
    definition: {method: tag, value: daily}
  - name: daily_alias
    definition: {method: selector, value: daily}
```

## Resource-name and package ref resolution

Core 1.11.0 offers two project behavior flags:

```yaml
flags:
  require_unique_project_resource_names: true
  require_ref_searches_node_package_before_root: true
```

`require_unique_project_resource_names` restores an error for duplicate node
names within a project. `require_ref_searches_node_package_before_root` makes an
ambiguous package-internal `ref()` search the referencing node's package before
the root project.

## Function selection, retry, and unit-test setup

Managed functions can be selected as a resource type or by name:

```bash
dbt list --select "resource_type:function"
dbt build --select "resource_type:function"
dbt build --select is_positive_int
```

All overloaded signatures share one DAG node and are selected and built
together. `dbt retry` reruns only overloads that failed.

Unit tests do not implicitly create warehouse functions. Build the function and
the tested model's ancestors first:

```bash
dbt build --select "+my_model_to_test" --empty
```
