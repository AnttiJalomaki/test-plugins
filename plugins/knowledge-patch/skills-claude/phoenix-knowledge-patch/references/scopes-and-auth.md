# Scoped Data Access and Authentication

This reference organizes the generator-scope and authentication guidance attributed to the `1.8-guides` batch.

## Generated scopes are authorization boundaries

Running:

```console
$ mix phx.gen.auth Accounts User users
```

generates an `Accounts.Scope` struct. Unless another default scope is already configured, it also registers a default user scope.

Generated browser authentication uses `fetch_current_scope_for_user` to assign the scope as `:current_scope`. The generated LiveView authentication setup includes a corresponding mount hook. Generated controllers and LiveViews then pass the scope as the first argument to context operations.

Once a default scope exists, all of these generators produce ownership fields and scoped queries instead of unqualified data access:

- `phx.gen.schema`
- `phx.gen.context`
- `phx.gen.live`
- `phx.gen.html`
- `phx.gen.json`

A generated context function follows this shape:

```elixir
def list_posts(%Scope{} = scope) do
  Repo.all(from post in Post, where: post.user_id == ^scope.user.id)
end
```

Put generated authenticated LiveView routes inside the authenticated `live_session`. Otherwise the mount hook does not establish the expected scope before the generated operations run.

## Declaring the default scope

Scopes are discovered from application configuration. There may be only one default scope.

```elixir
config :my_app, :scopes,
  user: [
    default: true,
    module: MyApp.Accounts.Scope,
    assign_key: :current_scope,
    access_path: [:user, :id],
    schema_key: :user_id,
    schema_type: :id,
    schema_table: :users,
    test_data_fixture: MyApp.AccountsFixtures,
    test_setup_helper: :register_and_log_in_user
  ]
```

The options have distinct responsibilities:

- `module` identifies the scope struct module.
- `assign_key` names the request and socket assign.
- `access_path` tells generated code how to read the owner's identifier from the scope.
- `schema_key` defines the generated ownership column.
- `schema_type` defines the schema field type.
- `schema_table` defines the referenced owner table.
- `test_data_fixture` supplies generated scope data.
- `test_setup_helper` prepares authentication and scope state for generated tests.

The fixture module must implement `<name>_scope_fixture/0`; for the `user` scope, that is `user_scope_fixture/0`. The configured setup helper must be importable by the generated controller or LiveView tests.

Use `schema_migration_type` to override the generated migration's ownership-column type without changing the schema type. Use `schema_table: nil` when the ownership value should be a plain scope-id column rather than a foreign key.

## Multiple scopes

An application may configure several scopes. Select a non-default scope with the generator's `--scope` option.

```elixir
config :my_app, :scopes,
  organization: [
    module: MyApp.Accounts.Scope,
    assign_key: :current_scope,
    access_path: [:organization, :id],
    route_prefix: "/organizations/:org",
    route_access_path: [:organization, :slug],
    schema_key: :org_id,
    schema_type: :id,
    schema_table: :organizations,
    test_data_fixture: MyApp.AccountsFixtures,
    test_setup_helper: :register_and_log_in_user_with_org
  ]
```

```console
$ mix phx.gen.live Blog Post posts title:string body:text --scope organization
```

`route_prefix` nests the generated routes. `route_access_path` supplies the URL-facing route value, such as an organization slug, independently from the identifier used by `access_path` and `schema_key` for data ownership.

## Route-derived scopes

Do not look up a nested organization without an existing authorization boundary merely because its slug came from a route.

Instead:

1. Start with the authenticated user scope.
2. Load the requested organization through that user scope.
3. Update `:current_scope` to include the organization in a browser plug.
4. Mirror that update in a LiveView `on_mount` hook.

The plug and mount hook keep controller and LiveView behavior aligned. Loading through the existing user scope keeps nested lookups authorization-aware, while `route_access_path` allows generated paths to use the configured slug.

## Magic-link-first generated authentication

The authentication generator uses email confirmation and magic-link login by default. Password authentication is opt-in, and new registration no longer collects a password.

Generated `UserAuth` includes:

- `fetch_current_scope_for_user`, which establishes the current scope.
- `require_authenticated_user`, which must run after the scope-fetching plug.
- `require_sudo_mode`, which requires recent authentication for sensitive operations.

Preserve that plug order when changing browser pipelines. For LiveView routes, preserve the equivalent authenticated `live_session` and mount-hook arrangement.

## Migrating authentication generated before 1.8

Generators do not retroactively update an application's copied authentication code. A full move from the password-registration flow therefore needs an explicit data migration and deliberate deployment plan.

Create a new migration that makes `hashed_password` nullable. Do not edit a migration that may already have run in other environments.

Set `hashed_password` to `nil` for every account that is still unconfirmed to prevent credential pre-stuffing.

That cleanup can invalidate a password chosen by a legitimate user who registered moments before deployment but has not confirmed yet. Mitigate the race in one of two ways:

- Deploy the migration during a low-traffic period.
- Introduce magic links without immediately removing the existing password-registration flow.

## Authentication assets

Generated authentication features assume that `phoenix_html.js` is part of the JavaScript bundle. `phx.gen.auth` warns when esbuild is unavailable.
