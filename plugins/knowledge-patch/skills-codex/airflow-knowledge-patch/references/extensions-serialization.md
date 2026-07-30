# Extensions, serialization, and XCom

## Plugin registration and imports

As of 3.0.0, plugins do not register operators, sensors, hooks, or executors.
They are plain Python classes; import them directly from their package instead
of using Airflow's plugin namespace:

```python
from my_plugin import MyHook
```

Legacy plugins that expose `appbuilder_views`, `appbuilder_menu_items`, or
`flask_blueprints` need the FAB provider compatibility layer or must move to
`external_views`, `fastapi_apps`, and `fastapi_root_middlewares`.

## React and navigation extensions

AIP-68 React Apps are an experimental 3.1.0 plugin surface for full
applications and dashboard/menu integrations in the modern UI. Backend
plugins also gain `iframe_views` for external views in navigation and Dag
pages.

In 3.3.0, plugin navigation can set `nav_top_level`. `/auth` and `/pluginsv2`
are reserved route prefixes. Owner-link and extra-link `href` values are
restricted to HTTP, HTTPS, `mailto`, or relative URLs.

`task_instance_mutation_hook` now receives the associated `DagRun`.
`BaseTrigger.on_kill()` is the extension hook for user actions against a
trigger.

## Operator extra links

The UI no longer executes custom operator-link code in 3.0.0. A custom
`BaseOperatorLink` declares an `xcom_key`; task execution stores the complete
URL under that key in XCom, and task-detail views retrieve it from the XCom
backend.

## Dag and Task SDK serialization

Dags are always JSON-serialized in 3.0.0. Every custom object embedded in a
Dag must have a JSON-compatible representation; Dag pickling is removed.

Airflow 3.1.0 establishes a versioned Dag-serialization contract in the Task
SDK, allowing separately deployed components to upgrade with less
coordination. It is a decoupling foundation, not complete code separation;
that fuller separation was planned for 3.2.

Serialization moves into the Task SDK in 3.2.0:

| Deprecated import | Supported import |
| --- | --- |
| `airflow.serialization.serde` | `airflow.sdk.serde` |
| `airflow.serialization.serializers.*` | `airflow.sdk.serde.serializers.*` |

The old imports warn and remain only until Airflow 4. Migrate custom
serializers now.

The deserializer interface receives the loaded class rather than a class-name
string as of 3.1.0:

```python
def deserialize(cls: type, version: int, data: Any):
    ...
```

`get_task_group_children_getter` and `task_group_to_dict` are no longer public
from `airflow.sdk.definitions.taskgroup`; they moved to server-side API
services in 3.1.0.

`PriorityWeightStrategy.serialize()` and `.deserialize()` are removed in
3.2.0.

## XCom semantics and safety

An unqualified `ti.xcom_pull(key="shared_state")` searches only the current
task after the Airflow 3 migration. Always name another producer:

```python
value = ti.xcom_pull(task_ids="upstream_task", key="shared_state")
```

XCom pickling is removed. Use JSON-native values or a custom XCom backend for
another representation.

In 3.1.0, `enable_xcom_deserialize_support` is removed. The API server does not
deserialize unknown Python objects merely to display them; it renders
non-native values through safer representations. `XCom.set()` and `XCom.get()`
reject empty keys.

In 3.2.0, `BaseXcom` is exported from `airflow.sdk`, and the UI can add, edit,
and delete XCom values.

In 3.3.0, async tasks have async XCom accessors. Structured outputs can
round-trip as Pydantic model instances when their types are registered from
the worker-side Dag. The XCom Execution API is team-scoped in multi-team
deployments.

## Parser statistics

`FileLoadStat` adds nullable `bundle_path` and `bundle_name` in 3.2.0. Paths
are real relative paths and no longer start with `/` to mean “relative to the
Dags folder.” Custom parsing tools should handle them as `pathlib.Path`
objects instead of depending on the old string convention.

## Provider hook deprecations

Provider connection-form hooks `get_connection_form_widgets` and
`get_ui_field_behaviour` are deprecated in 3.2.0. Do not start new UI
integrations on these methods; use the supported provider/API extension
surfaces.
