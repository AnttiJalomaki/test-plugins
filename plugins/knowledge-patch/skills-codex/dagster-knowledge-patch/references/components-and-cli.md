# Components and CLI

## Component authoring

Components are production-ready and recommended for new projects (1.11.0).
Declare definitions in `defs.yaml` or implement typed Python `Component`
subclasses. Decorate Python helpers with `@template_var` to expose them to
YAML. `build_defs_for_component` builds definitions outside a `defs` folder.
Templates can compose Components through `load_component_at_path` and
`build_defs_at_path` for code that still targets that API generation:

```yaml
deps:
  - "{{ load_component_at_path('dbt_ingest').asset_key_for_model('customers') }}"
```

For the current `ComponentLoadContext` names, use `load_component` and
`build_defs`; see the upgrade reference for the compatibility-method removal.

## Template scopes

As of 1.12.0, templates expose:

- automation conditions, partition definitions, and `FreshnessPolicy` through
  `dg`;
- loading helpers and `project_root` through `context`;
- `datetime` and `timedelta` through `datetime`.

```jinja
{{ dg.AutomationCondition.on_missing() & dg.AutomationCondition.in_latest_time_window() }}
{{ dg.DailyPartitionsDefinition("2025-01-01") }}
{{ context.load_component("warehouse") }}
```

## State-backed Components

`StateBackedComponent` separates persisted discovery state from YAML or Python
configuration (1.12.0). It supports local state, versioned storage, and
code-server snapshots. Airbyte, Fivetran, Power BI, Airflow, and dbt project
Components use the model, and generated deployment workflows refresh their
state.

In 1.13.0, state-backed integrations such as Airbyte and Fivetran default to
`LOCAL_FILESYSTEM` rather than `legacy_code_server_snapshots`. Configure
storage explicitly if code-server snapshot behavior is required.

## Core `dg` workflow

The stable `dg` CLI covers the project lifecycle (1.11.0):

- `dg scaffold` creates definitions and project objects.
- `dg dev` starts local development.
- `dg launch` launches work.
- `dg list` inspects definitions.
- `dg check` validates configuration and definitions.
- `dg utils` hosts utilities.

`create-dagster project` creates the modern `src/` plus `defs/` layout, sets
up local `dg`, and does not require an active Python environment.

Additional commands include `dg list component-tree`, `dg check toml`,
`dg mcp`, `dg api secret list`, and `dg api secret get`. Validation of
`requirements.env` is opt-in for `dg check yaml`. MCP dependencies live in
the `dagster-dg-cli` `ai` extra rather than the base CLI.

## Definition-querying APIs

The 1.12.0 `dg api` surface includes:

- `schedule list` and `schedule get`;
- `job list` and `job get`;
- `asset-check list` and `asset-check get-executions`;
- `asset get-partition-status`.

In 1.13.0, `dg api run launch` launches through the Dagster+ API.

## Deployment scaffolding

`dg scaffold build-artifacts` generates Docker and deployment configuration
for ECR, DockerHub, GHCR, ACR, or GCR (1.12.0).
`dg scaffold github-actions` generates Serverless- or Hybrid-aware CI.
Generated GitHub Actions workflows also refresh state-backed Component state
during deployment.

`dg plus deploy configure` prepares an existing project for Dagster+ and can
scaffold GitLab CI. Use `dg plus deploy start` for deployment validation and
deployment startup.

## Dagster+ CLI configuration

For EU authentication, use `dg plus login --region eu`. Inspect active CLI
configuration with `dg plus config view`.

`[tool.dg.project]` accepts `agent_queue` and `image` to populate generated
`dagster_cloud.yaml`. `dg plus pull env` merges secrets into an existing
`.env` without replacing locally defined values.

Values of `DG_PROJECT_PYTHON_EXECUTABLE` in a project `.env` follow
`python-dotenv` syntax in 1.13.0, including `export`, quoting, and trailing
comments.

## Development database controls

`dg dev` and `dagster dev` accept database-pool controls in 1.12.0,
including `--db-pool-recycle` and `--db-pool-pre-ping`.
