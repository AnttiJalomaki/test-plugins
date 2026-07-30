# Components and CLI

Use this reference for component authoring, templating, state management,
scaffolding, and project or Dagster+ CLI workflows.

## Authoring Components

Components are production-ready and the recommended default for new projects
as of 1.11.0. Declare them in `defs.yaml` or implement typed Python
`Component` subclasses. Decorate Python helpers with `@template_var` to expose
them to YAML.

Use `build_defs_for_component` when a component is outside a `defs` folder.
Templates can compose components with loading helpers:

```yaml
deps:
  - "{{ load_component_at_path('dbt_ingest').asset_key_for_model('customers') }}"
```

That example reflects the 1.11.0 helper name. Current component contexts use
`context.load_component(...)` and `context.build_defs(...)`; see the upgrade
reference before copying older templates.

Integration Components for Airbyte, Fivetran, Power BI, Sling, and dlt expose
an overridable `get_asset_spec`. Airbyte and Fivetran also expose `execute`,
and `DbtProjectComponent` exposes `get_asset_spec` and
`get_asset_check_spec` (1.11.0). Airbyte and Fivetran Components no longer
reserve the former `io_manager`, `airbyte`, or `fivetran` resource keys.

## State-backed Components

`StateBackedComponent` separates persisted discovery state from YAML or
Python configuration (1.12.0). It supports local state, versioned storage,
and code-server snapshots. Airbyte, Fivetran, Power BI, Airflow, and dbt
project Components use this model. Generated GitHub Actions deployment
workflows refresh state during deployment.

As of 1.13.0, state-backed integrations such as Airbyte and Fivetran default
to `LOCAL_FILESYSTEM` instead of `legacy_code_server_snapshots`. Configure a
different store when code-location filesystems are ephemeral or not shared.

## Template scopes

Component templates expose these namespaces as of 1.12.0:

- `dg`: automation conditions, partition definitions, and `FreshnessPolicy`
- `context`: loading helpers and `project_root`
- `datetime`: `datetime` and `timedelta`

```jinja
{{ dg.AutomationCondition.on_missing() & dg.AutomationCondition.in_latest_time_window() }}
{{ dg.DailyPartitionsDefinition("2025-01-01") }}
{{ context.load_component("warehouse") }}
```

## Core `dg` workflows

The stable `dg` CLI introduced in 1.11.0 unifies common work:

| Goal | Command family |
| --- | --- |
| Create definitions and files | `dg scaffold` |
| Start local UI and services | `dg dev` |
| Launch work | `dg launch` |
| Inspect definitions | `dg list` |
| Validate configuration | `dg check` |
| Use auxiliary tools | `dg utils` |

`create-dagster project` creates the modern `src/` plus `defs/` layout with a
local `dg` setup and does not require an active Python environment (1.11.0).

Additional 1.11.0 tools include:

- `dg list component-tree`
- `dg check toml`
- `dg mcp`
- `dg api secret list`
- `dg api secret get`

Validation of `requirements.env` is opt-in for `dg check yaml`. MCP-related
dependencies are isolated in the `dagster-dg-cli` `ai` extra rather than the
base CLI.

## Definition-querying APIs

The `dg api` surface added these commands in 1.12.0:

- `schedule list` and `schedule get`
- `job list` and `job get`
- `asset-check list` and `asset-check get-executions`
- `asset get-partition-status`

In 1.13.0, `dg api run launch` can launch runs through the Dagster+ API.

## Scaffolding deployment artifacts

`dg scaffold build-artifacts` creates Docker and deployment configuration for
ECR, DockerHub, GHCR, ACR, or GCR (1.12.0). `dg scaffold github-actions`
generates Serverless- or Hybrid-aware CI.

For an existing project, `dg plus deploy configure` prepares Dagster+
deployment configuration and can scaffold GitLab CI. Use
`dg plus deploy start` in place of deprecated `dagster-cloud ci check` so the
deployment is validated as it starts.

## Dagster+ login and project settings

- Use `dg plus login --region eu` for EU authentication (1.12.0).
- Use `dg plus config view` to inspect active CLI configuration (1.12.0).
- `[tool.dg.project]` accepts `agent_queue` and `image` for generated
  `dagster_cloud.yaml` (1.12.0).
- `dg plus pull env` merges secrets into an existing `.env`; it does not
  replace values already defined locally (1.12.0).
- Values of `DG_PROJECT_PYTHON_EXECUTABLE` in a project `.env` follow
  `python-dotenv` syntax, including `export`, quoting, and trailing comments
  (1.13.0).

## Configuration types and secrets

Configurable resource fields support union annotations such as `Foo | Bar`
since 1.11.0.

Hide a resource parameter in the UI by marking its Pydantic field as secret
(1.12.0):

```python
from pydantic import Field

token: str = Field(json_schema_extra={"dagster__is_secret": True})
```

Date-like YAML strings such as `"2021-10-30"` remain strings instead of being
converted to datetimes as of 1.13.0.

