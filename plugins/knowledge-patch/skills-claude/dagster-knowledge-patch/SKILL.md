---
name: dagster-knowledge-patch
description: Dagster
version: 1.13.0
license: MIT
metadata:
  author: Nevaberry
---


# Dagster Knowledge Patch

Use this skill when changing Dagster definitions, Components, automation,
execution, deployment, or integrations. Start with the migration checks below,
then open the topic reference that matches the work. Prefer the project's
installed packages, definitions, and validated behavior when they differ from
examples.

## Reference index

| Reference | Topics |
| --- | --- |
| [Upgrade and compatibility](references/upgrade-and-compatibility.md) | Removed and renamed APIs, freshness migration, component loading, runtime and package requirements |
| [Components and CLI](references/components-and-cli.md) | Components, templates, state, scaffolding, `dg`, project configuration, secrets |
| [Assets and automation](references/assets-and-automation.md) | Asset values, selections, partitions, freshness, checks, ownership, metadata |
| [Execution and operations](references/execution-and-operations.md) | Coordination, pools, backfills, executors, GraphQL, daemon behavior, limits |
| [Integrations](references/integrations.md) | dbt, Airbyte, Fivetran, Databricks, BI, cloud, Pipes, and other packages |
| [Deployment and storage](references/deployment-and-storage.md) | Helm, Kubernetes, ECS, authentication, databases, IO managers, runtime support |

## Migration triage

Before changing code, search for these high-impact compatibility points:

```text
include_sources
@experimental
FreshnessPolicy
dagster.preview.freshness
load_defs
load_component_at_path
build_defs_at_path
external_asset_from_spec
external_assets_from_specs
get_all_asset_specs
legacy_freshness_policy
auto_observe_interval_minutes
build_airbyte_assets
dagster project
dagster-cloud ci check
dg docs integrations
dg utils integrations
DAGSTER_MAX_ASSET_BACKFILL_RETRIES
```

### Runs need the daemon by default

The queued coordinator is the default. A deployment without a running Dagster
daemon can accept a run without launching it. Restore immediate in-process
launching only when that behavior is intentional:

```yaml
run_coordinator:
  module: dagster.core.run_coordinator.sync_in_memory_run_coordinator
  class: SyncInMemoryRunCoordinator
```

Pool-aware run blocking is also enabled by default. At op granularity, a run
can leave the queue when one op is runnable; at run granularity, every pool the
run uses must have capacity.

### Use the current external-asset forms

Replace `include_sources` with `include_external_assets`. Do not call the
removed `external_asset_from_spec` or `external_assets_from_specs` helpers;
place `AssetSpec` values directly in `Definitions` or construct an
`AssetsDefinition`.

```python
from dagster import AssetSpec, Definitions

defs = Definitions(assets=[AssetSpec("upstream_external")])
```

Pass `deps` as a sequence, even for one dependency. Use `AssetDep` when a
dependency carries configuration such as a partition mapping.

```python
from dagster import AssetDep, asset

@asset(deps=[AssetDep("upstream")])
def downstream():
    ...
```

Call `Definitions.resolve_all_asset_specs()`; the former
`get_all_asset_specs()` method is removed.

### Complete the freshness migration

Import `FreshnessPolicy` and `apply_freshness_policy` from top-level `dagster`.
The intermediate preview import and the older legacy policy parameters are not
current APIs. Replace legacy observation intervals with an
`automation_condition` and schedule- or sensor-driven automation.

Freshness evaluation runs automatically. To opt out:

```yaml
freshness:
  enabled: false
```

Use `AutomationCondition.freshness_passed()`, `freshness_warned()`, and
`freshness_failed()` when downstream automation depends on evaluation state.

### Update component loading

For a definitions folder, use `load_from_defs_folder(path)` instead of the
deprecated non-public `load_defs`. In templates and component code, call
`context.load_component(...)` and `context.build_defs(...)`; compatibility
methods ending in `_at_path` have been removed.

New projects should prefer Components and the `src/` plus `defs/` scaffold.
Use `create-dagster project` for project creation and `dg` for scaffolding,
development, validation, listing, launches, utilities, and Dagster+ workflows.

### Check runtime and database dependencies

Python 3.10 is the minimum after Python 3.9 support was dropped. Core packages
and most libraries support Python 3.14; deployment tooling also supports modern
Python versions as detailed in the runtime reference.

MySQL deployments need `dagster instance migrate` for the `LongText` changes.
PostgreSQL users must declare `psycopg2-binary` themselves if they need it,
because `dagster-postgres` no longer installs it transitively.

## High-value current patterns

### Return and dynamically load asset values

`MaterializeResult(value=...)` sends a value through the asset IO manager and
supports a generic result type. A downstream asset can dynamically load it
without declaring it as a function parameter:

```python
import dagster as dg

@dg.asset
def upstream() -> dg.MaterializeResult[int]:
    return dg.MaterializeResult(value=42)

@dg.asset(deps=[upstream])
def downstream(context: dg.AssetExecutionContext):
    return context.load_asset_value(dg.AssetKey("upstream"))
```

### Model derived data as virtual assets

Use preview `is_virtual=True` for views and derived tables that reflect
upstream changes without explicit materialization. Virtual assets influence
staleness, execution planning, and declarative automation.

```python
import dagster as dg

reporting_view = dg.AssetSpec("reporting_view", is_virtual=True)
```

### Use unified selection expressions

Selection expressions combine lineage, attributes, and Boolean operators.
They are shared across Components YAML, the Asset Catalog, saved selections,
alerts, and insights. Useful current selectors include:

```text
sensor:daily_refresh
schedule:hourly_load
job:analytics
automation_type:schedule
is:materializable
partitions:"static"
group:"marketing/*"
```

Schedule and sensor selectors include assets in targeted jobs as well as assets
selected directly by the instigator. Group names may contain `/` and render as
a hierarchy.

### Treat integration discovery state as local by default

State-backed integration Components separate discovery state from YAML or
Python configuration. Their default storage is now `LOCAL_FILESYSTEM`, not
legacy code-server snapshots. Configure storage explicitly when local state is
not durable or shared enough for the deployment.

### Follow GraphQL cursors

`logsForRun` and `eventConnection` return at most 1,000 events by default.
Always continue with the returned cursor. For webservers under a URL prefix,
set `DagsterGraphQLClient(path_prefix=...)`; bounded run-selection previews
expose both a limit and the true asset and check counts.

### Handle custom executor initialization failures

A process-based custom executor must turn each resource-initialization failure
into a failure-or-retry event, yield it, and register it with active execution.
Skipping registration can strand the run in `started` state. See the execution
reference for the required event sequence.

## Working method

1. Inspect installed Dagster and integration package versions, Python version,
   `dagster.yaml`, component state storage, and deployment charts.
2. Search for removed names from migration triage before adding new behavior.
3. Open the narrowest topic reference and apply all coupled changes; many API
   migrations cross definitions, YAML, generated deployment files, and CI.
4. Run `dg check yaml` when relevant, `dg check toml`, definition validation,
   and the project's tests. Remember that `requirements.env` checks are opt-in.
5. For GraphQL, daemon, backfill, or integration changes, verify pagination,
   cancellation, retry, and cleanup behavior—not only successful execution.

