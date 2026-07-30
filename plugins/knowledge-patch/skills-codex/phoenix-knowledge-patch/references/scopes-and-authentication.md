# Scopes and Authentication

The generator guidance in this file comes from batch `1.8-guides`.

## Generated Scopes Are Authorization Boundaries

Running:

```console
$ mix phx.gen.auth Accounts User users
```

creates an `Accounts.Scope` struct. Unless the application already has another
default, the generator also registers a default user scope.
`fetch_current_scope_for_user` assigns that scope as `:current_scope` for
browser requests, and generated LiveView support installs a corresponding mount
hook. Generated controllers and LiveViews then pass the scope as the first
argument to context operations.

```elixir
def list_posts(%Scope{} = scope) do
  Repo.all(from post in Post, where: post.user_id == ^scope.user.id)
end
```

After a default scope is configured, these generators produce ownership fields
and scoped queries instead of unqualified data access:

- `phx.gen.schema`
- `phx.gen.context`
- `phx.gen.live`
- `phx.gen.html`
- `phx.gen.json`

Generated authenticated LiveView routes belong inside the authenticated
`live_session`; otherwise the expected scope has not been mounted before the
generated operations run.

## Declare a Generator Scope

Generators discover scopes from application configuration. Only one scope may
be marked as the default. `access_path` tells generated code how to reach the
owner identifier, while the `schema_*` keys describe the generated ownership
column and association.

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

The configured fixture module must export `<name>_scope_fixture/0`, such as
`user_scope_fixture/0`. The setup helper must be importable by generated
controller or LiveView tests.

Use `schema_migration_type` to override the type used for the migration column.
Set `schema_table: nil` when the generated schema should contain a plain scope
identifier rather than a foreign key.

## Use Multiple and Route-Aware Scopes

An application may declare several scopes. Select a non-default scope with
`--scope`:

```console
$ mix phx.gen.live Blog Post posts title:string body:text --scope organization
```

`route_prefix` nests generated routes. `route_access_path` identifies the value
placed in the URL, such as an organization slug, independently of the database
ownership key in `access_path`.

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

For a route-derived organization scope, first load the organization through the
existing user scope. Then update `:current_scope` in both a browser plug and a
LiveView `on_mount` hook. This keeps nested lookup authorization-aware while
letting generated paths use the configured slug.

## Magic-Link-First Authentication

The auth generator now uses email confirmation and magic-link login as its
primary flow. Password authentication is opt-in, and new registration no longer
collects a password.

Generated `UserAuth` provides:

- `fetch_current_scope_for_user` to establish the request scope.
- `require_authenticated_user` to require a signed-in user. Run it after the
  scope-fetching plug.
- `require_sudo_mode` for sensitive operations that require recent
  authentication.

## Migrate Older Generated Authentication

Generated auth modules, templates, and migrations are application code; a
Phoenix dependency upgrade does not rewrite them. To migrate a password-first
registration flow:

1. Add a new migration that makes `hashed_password` nullable. Do not alter an
   already-used historical migration.
2. Set `hashed_password` to `nil` for every account that is still unconfirmed.
   This avoids credential pre-stuffing through an unconfirmed registration.
3. Account for the race in which a person has just registered and selected a
   password that the cleanup would invalidate.

Deploy the cleanup during low traffic, or introduce magic links while keeping
the existing password flow until the transition can be coordinated safely.

