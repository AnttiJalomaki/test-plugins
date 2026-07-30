# Upgrade and Compatibility

Use this reference while migrating imports, definitions, component APIs, or
project tooling. Entries are grouped by the surface being changed.

## Asset and definition API migrations

### External assets and selections

- Since 1.10.0, `AssetSelection` APIs use `include_external_assets`; rename the
  earlier `include_sources` keyword.
- In 1.13-upgrade, `external_asset_from_spec` and
  `external_assets_from_specs` were removed. Pass `AssetSpec` objects directly
  to `Definitions`, or construct an `AssetsDefinition` from them.
- In 1.13-upgrade, `deps` stopped accepting a single `AssetKey`. Always pass a
  sequence; use `AssetDep` when attaching a partition mapping or other
  dependency configuration.
- In 1.13-upgrade, replace removed
  `Definitions.get_all_asset_specs()` with
  `Definitions.resolve_all_asset_specs()`.

```python
from dagster import AssetDep, AssetSpec, Definitions, asset

defs = Definitions(assets=[AssetSpec("external_source")])

@asset(deps=[AssetDep("external_source")])
def downstream():
    ...
```

Definition validation became stricter in 1.11.0: invalid partition mappings,
including time-partitioned dependencies whose time zones differ, fail
`dagster definitions validate`. `Definitions` and `AssetsDefinition` also
reject distinct `AssetSpec` values with the same asset key.

## Freshness migration

The freshness API changed in stages; migrate to the final state rather than
stopping at an intermediate compatibility alias.

- In 1.11.0, the former `FreshnessPolicy` became
  `LegacyFreshnessPolicy`. Importing `FreshnessPolicy` from top-level
  `dagster` errored at that stage; a deprecated alias was available from
  `dagster.deprecated`. The `freshness_policy` field on assets, specs, and
  `ResolvedAssetSpec` resolvers began carrying the new policy type.
- In 1.12-upgrade, `FreshnessPolicy` and `apply_freshness_policy` moved out of
  preview and became top-level `dagster` exports. Replace imports from
  `dagster.preview.freshness`.
- In 1.12.0, the `build_.*_freshness_checks` helpers were superseded by
  freshness policies. dbt and Sling translators removed
  `get_freshness_policy` and stopped parsing legacy policies from integration
  configuration.
- In 1.13-upgrade, `legacy_freshness_policy` and
  `auto_observe_interval_minutes` were removed from
  `@observable_source_asset`. Legacy freshness-policy parameters were also
  removed from `AssetsDefinition`, `AssetOut`, and `load_assets_from_*`.
  Replace them with `automation_condition` plus schedule- or sensor-based
  automation.

Use the current import:

```python
from dagster import FreshnessPolicy, apply_freshness_policy
```

The `FreshnessDaemon` evaluates policies by default as of 1.12-upgrade.
Disable automatic evaluation only when deliberate:

```yaml
freshness:
  enabled: false
```

## Component-loading migrations

- In 1.11.0, `load_defs` became deprecated and non-public. Use
  `load_from_defs_folder(path)`.
- Also in 1.11.0, Sling and Airflow Components replaced
  `asset_post_processors` with top-level `post_processors`.
  `SlingReplicationCollectionComponent` takes `connections` directly rather
  than the deprecated `sling` YAML field or Python `resource` argument.
- In 1.12.0, `ComponentLoadContext.build_defs_at_path` and
  `ComponentLoadContext.load_component_at_path` were renamed to `build_defs`
  and `load_component`, with compatibility methods temporarily retained.
- In 1.13-upgrade, those `_at_path` compatibility methods were removed. Use
  `context.build_defs(...)` and `context.load_component(...)`.

## Integration API removals

### Airbyte

In 1.13-upgrade, `AirbyteState` was removed; use
`AirbyteJobStatusType`. `build_airbyte_assets()` no longer accepts
`legacy_freshness_policy` or `auto_materialize_policy`.

### Looker, Power BI, and Sigma

In 1.13-upgrade, the following resource methods were removed:

- `DagsterLookerResource.build_defs`
- `PowerBIWorkspace.build_defs`
- `SigmaOrganization.build_defs`

Load specs with `load_looker_asset_specs`, `load_powerbi_asset_specs`, or
`load_sigma_asset_specs`, then pass them to `Definitions`. Loader translator
arguments require instances, not classes:

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

## CLI replacements

- Since 1.11.0, `create-dagster project` supersedes
  `dagster project scaffold`.
- In 1.12.0, all `dagster project` commands were removed. Use
  `create-dagster` for project creation and `dg` for project tasks.
- The removed `dg docs integrations` command became
  `dg utils integrations` in 1.12.0; `dg utils integrations` was itself
  removed in 1.13.0.
- `dagster-cloud ci check` is deprecated as of 1.12.0. Use
  `dg plus deploy start`, which also performs deployment validation.

## API lifecycle and package status

Since 1.10.0, lifecycle markers distinguish `@preview`, `@beta`, and
`@superseded` APIs, including matching annotations and warnings. The
`@experimental` decorator, annotations, and warnings were removed; choose
preview or beta status explicitly.

`dagster-blueprints` was removed in 1.10.0, with Components as its conceptual
successor. `dagster-sdf` moved to the community-supported repository.

## Language and library compatibility

- Dagster 1.11.0 added Python 3.13 and protobuf 6.x support and removed its
  Click `<8.2` cap.
- In 1.11.0, `dagster-deltalake` and `dagster-deltalake-polars` began
  requiring `deltalake>=1.0.0` without user-facing API changes.
- In 1.12.0, Python 3.9 support was dropped, so Python 3.10 is the minimum.
  Core Dagster and most libraries support Python 3.14; `dg plus deploy`
  supports Python 3.13 and 3.14.
- In 1.13.0, `dagstermill` requires `papermill>=2.0.0` and defaults the
  Jupyter kernel-startup timeout to 120 seconds rather than 60.
- In 1.13.0, `dagster-airlift` supports Python 3.12, 3.13, and 3.14.
