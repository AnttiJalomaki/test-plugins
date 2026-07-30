# Upgrades and API migration

## Coordination, pools, and selection names

Since 1.10.0, the queued run coordinator is the default, so the daemon must be
running for runs to launch. Restore immediate in-process behavior only by
configuring
`dagster.core.run_coordinator.sync_in_memory_run_coordinator.SyncInMemoryRunCoordinator`.
Concurrency-key and pool run blocking is also on by default. Op granularity
dequeues a run when at least one op can execute; run granularity waits until
every pool used by the run has a slot.

At first, pool names allowed only letters, numbers, dashes, and underscores,
and `dagster-dbt`, `dagster-dlt`, and `dagster-sling` gained pool support.
Since 1.12.0, any non-whitespace character is accepted.

Also since 1.10.0, replace the `include_sources` keyword on `AssetSelection`
APIs with `include_external_assets`.

## API lifecycle

The lifecycle decorators distinguish `@preview`, `@beta`, and `@superseded`,
with matching annotations and warnings (1.10.0). `@experimental` and its
annotations and warnings were removed; classify the API as preview or beta.

The `dagster-blueprints` package was removed, with Components as its
conceptual successor. `dagster-sdf` moved to the community-supported
repository.

## Freshness migration

The former `FreshnessPolicy` became `LegacyFreshnessPolicy` in 1.11.0.
Importing that old policy as `FreshnessPolicy` from top-level `dagster`
errors. Legacy code can temporarily import its deprecated alias from
`dagster.deprecated`. Asset and spec `freshness_policy` fields use the new
policy type, and `ResolvedAssetSpec` resolvers can set it.

In 1.12-upgrade, the current `FreshnessPolicy` and
`apply_freshness_policy` moved from `dagster.preview.freshness` to top-level
exports:

```python
from dagster import FreshnessPolicy, apply_freshness_policy
```

The `FreshnessDaemon` now evaluates policies by default. Disable evaluation
only when intended:

```yaml
freshness:
  enabled: false
```

In 1.12.0, freshness policies superseded the
`build_.*_freshness_checks` helpers. dbt and Sling translators removed
`get_freshness_policy` and stopped parsing legacy policies from integration
configuration. Automation may branch on the newest evaluation through
`AutomationCondition.freshness_passed()`, `freshness_warned()`, and
`freshness_failed()`.

In 1.13-upgrade, `@observable_source_asset` removed
`legacy_freshness_policy` and `auto_observe_interval_minutes`. Legacy policy
arguments were also removed from `AssetsDefinition`, `AssetOut`, and
`load_assets_from_*`. Use `automation_condition` with schedule- or
sensor-based automation.

## Component API migrations

In 1.11.0:

- `load_defs` became deprecated and non-public; use
  `load_from_defs_folder(path)`.
- Sling and Airflow Components removed `asset_post_processors`; put
  `post_processors` at the top level.
- `SlingReplicationCollectionComponent` accepts `connections` directly, not
  the deprecated `sling` YAML field or Python `resource` argument.

In 1.12.0, `ComponentLoadContext.build_defs_at_path` and
`load_component_at_path` were renamed to `build_defs` and `load_component`,
with compatibility methods retained temporarily. In 1.13-upgrade, those
compatibility methods were removed. Call `context.build_defs(...)` and
`context.load_component(...)`.

## Removed and renamed asset APIs

The following changes apply in 1.13-upgrade:

- `external_asset_from_spec` and `external_assets_from_specs` were removed.
  Put `AssetSpec` objects directly in `Definitions`, or build an
  `AssetsDefinition`.
- `deps` no longer accepts one bare `AssetKey`; pass a sequence. Use
  `AssetDep` for partition mappings or other dependency configuration.
- `Definitions.get_all_asset_specs()` was removed; call
  `Definitions.resolve_all_asset_specs()`.

```python
from dagster import AssetDep, AssetSpec, Definitions, asset

defs = Definitions(assets=[AssetSpec("my_asset")])

@asset(deps=[AssetDep("upstream")])
def downstream(): ...
```

## Integration removals

The Airbyte `AirbyteState` type was removed in 1.13-upgrade; use
`AirbyteJobStatusType`. `build_airbyte_assets()` no longer accepts
`legacy_freshness_policy` or `auto_materialize_policy`.

`DagsterLookerResource.build_defs`, `PowerBIWorkspace.build_defs`, and
`SigmaOrganization.build_defs` were removed. Load specs with
`load_looker_asset_specs`, `load_powerbi_asset_specs`, or
`load_sigma_asset_specs` and pass them to `Definitions`. These loaders require
translator instances, not translator classes:

```python
defs = Definitions(
    assets=load_looker_asset_specs(
        looker_resource,
        dagster_looker_translator=MyTranslator(),
    )
)
```

Deprecated translator key helpers were removed. Implement
`get_asset_spec(...)` and derive keys from the returned spec.

## CLI and runtime migration

The stable `dg` command replaced several older entry points:

- `create-dagster project` superseded `dagster project scaffold` in 1.11.0.
- All `dagster project` commands were removed in 1.12.0.
- Removed `dg docs integrations` became `dg utils integrations`, which was
  itself removed in 1.13.0.
- `dagster-cloud ci check` is deprecated; use `dg plus deploy start`, which
  also performs deployment validation.

Python 3.9 support was dropped in 1.12.0, so require at least Python 3.10.
The core package and most libraries support Python 3.14; `dg plus deploy`
supports Python 3.13 and 3.14. This follows 1.11.0 support for Python 3.13
and protobuf 6.x, along with removal of the Click `<8.2` cap.

For database upgrades in 1.12.0:

- MySQL users must run `dagster instance migrate` for `LongText` changes to
  bulk-action bodies and cached asset status data.
- `dagster-postgres` no longer installs `psycopg2-binary` transitively;
  declare it in projects that use it.
