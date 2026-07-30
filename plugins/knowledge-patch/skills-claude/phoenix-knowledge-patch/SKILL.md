---
name: phoenix-knowledge-patch
description: Phoenix
version: 1.8.x
license: MIT
metadata:
  author: Nevaberry
---


# Phoenix Knowledge Patch

Use this skill when maintaining or generating Phoenix applications whose code may rely on current generator, authentication, routing, endpoint, channel, or JavaScript behavior.

Check the application's Phoenix dependency before applying version-specific advice. Prefer the application's configuration, generated code, tests, and observed behavior when they differ from this guidance.

## Reference index

| Reference | Topics |
| --- | --- |
| [references/scopes-and-auth.md](references/scopes-and-auth.md) | Generator scopes, ownership-aware contexts, nested route scopes, magic-link authentication, sudo mode, authentication migration |
| [references/runtime-security-and-routing.md](references/runtime-security-and-routing.md) | Runtime prerequisites, layouts, browser headers, controller and router deprecations, endpoint configuration, channels, LongPoll, verified routes, assigns |
| [references/generators-client-and-testing.md](references/generators-client-and-testing.md) | Generator CLI changes, auth assets, project tooling, JavaScript socket and Presence behavior, guarded channel assertions |

## Breaking changes and deprecations

### Declare controller formats

Every `use Phoenix.Controller` must provide `:formats`; use an empty list when the controller has no formats.

```elixir
use Phoenix.Controller, formats: [:html]
```

Replace deprecated controller configuration:

- Remove `:namespace` and `:put_default_views` options.
- Use module-qualified layouts.
- Move trailing-slash URL generation from the router's deprecated `:trailing_slash` option to `Phoenix.VerifiedRoutes`.

```elixir
put_layout(conn, html: {MyAppWeb.Layouts, :print})
```

### Read endpoint compile-time configuration explicitly

Do not use the former injected `config` variable inside `Phoenix.Endpoint`. Read compile-time settings with `Application.compile_env/3`.

```elixir
@value Application.compile_env(
  :my_app,
  [MyAppWeb.Endpoint, :some_setting],
  :default
)
```

Audit settings that exist only in runtime configuration: code that now reads them at compile time can otherwise fail during boot.

### Treat layouts as components

Generated applications use one `root.html.heex` around the render pipeline. Invoke dynamic layouts such as the application layout as function components from templates.

```heex
<Layouts.app flash={@flash}>
  ...
</Layouts.app>
```

Do not configure module-less layouts or expect an additional dynamic layout in the render pipeline.

### Respect stricter verified routes

Do not invoke `use Phoenix.VerifiedRoutes` more than once in a module, and do not interpolate a list into a verified route. Both forms raise in Phoenix 1.8.6.

For stable query strings in tests, opt into deterministic parameter ordering:

```elixir
config :phoenix, sort_verified_routes_query_params: true
```

### Enforce supported runtime and transport limits

Phoenix 1.8 requires Erlang/OTP 25 or later.

Phoenix 1.8.9 limits each transport to 100 channel processes by default through `max_channels_per_transport`. Raise the option deliberately when one client legitimately multiplexes more than 100 channels.

LongPoll is opt-in from Phoenix 1.8.2 onward. Account for its security fixes and 100-event request batch limit before enabling it for high-frequency traffic.

## Generated scopes and authorization boundaries

Treat a generated scope as a data-access boundary, not merely a convenient assign. Generated context functions accept the scope first and restrict queries by its owner.

```elixir
def list_posts(%Scope{} = scope) do
  Repo.all(from post in Post, where: post.user_id == ^scope.user.id)
end
```

After defining a default scope, these generators emit ownership fields and scoped queries:

- `phx.gen.schema`
- `phx.gen.context`
- `phx.gen.live`
- `phx.gen.html`
- `phx.gen.json`

Keep authenticated generated LiveView routes in the authenticated `live_session`; its mount hook must establish `:current_scope` before generated operations run.

### Configure one default scope

Declare scopes under the application `:scopes` configuration. Only one may be the default. The core ownership settings are:

- `module` and `assign_key` for the scope value.
- `access_path` for the owner's identifier.
- `schema_key`, `schema_type`, and `schema_table` for generated persistence.
- `test_data_fixture` and `test_setup_helper` for generated tests.

Use `schema_migration_type` when the migration column type differs from `schema_type`. Set `schema_table: nil` to generate a plain scope-id column instead of a foreign key.

The fixture module must expose `<name>_scope_fixture/0`, and generated controller or LiveView tests must be able to import the configured setup helper.

See [references/scopes-and-auth.md](references/scopes-and-auth.md) for complete configuration examples.

### Select and derive non-default scopes safely

Pass `--scope <name>` to select a non-default configured scope. `route_prefix` controls nested generated routes, while `route_access_path` may use a URL-facing value such as a slug independently from the database ownership key.

For route-derived organization scope:

1. Load the organization through the already-authorized user scope.
2. Replace `:current_scope` in a browser plug.
3. Perform the same replacement in a LiveView `on_mount` hook.

This preserves authorization for nested lookups while allowing generated paths to use the configured slug.

## Authentication quick reference

The generated authentication flow is magic-link-first:

- Registration no longer asks for a password.
- Email confirmation and magic-link login are generated by default.
- Password authentication is opt-in.
- `require_sudo_mode` protects sensitive actions by requiring recent authentication.
- `fetch_current_scope_for_user` must run before `require_authenticated_user`.

`mix phx.gen.auth Accounts User users` generates `Accounts.Scope`, registers a default user scope unless another default exists, assigns `:current_scope` for browser requests, and installs corresponding LiveView mounting behavior. Generated controllers and LiveViews pass the scope to context operations.

The generator expects `phoenix_html.js` in the JavaScript bundle and warns when esbuild is unavailable.

### Migrate older generated auth cautiously

Generated authentication code is not upgraded automatically.

For a full migration from password-at-registration:

1. Add a new migration that makes `hashed_password` nullable; do not edit an old migration.
2. Set `hashed_password` to `nil` for every account that remains unconfirmed to prevent credential pre-stuffing.
3. Address the race in which a newly registered user may lose a chosen password: deploy during low traffic or add magic links without immediately replacing the existing flow.

## Security and production defaults

When no Content Security Policy is supplied, `put_secure_browser_headers` sets:

```text
base-uri 'self'; frame-ancestors 'self';
```

Supply an explicit policy when third-party framing is intentional. Do not depend on the removed `x-download-options` or `x-frame-options` defaults.

Generated production configuration enables `force_ssl` by default.

Phoenix 1.8.7 masks a `token` parameter in logs by default alongside `password`. Phoenix 1.8.9 also hardens JavaScript Presence against keys matching `Object.prototype` members.

## Common generator workflows

The context argument is optional for `phx.gen.live`, `phx.gen.html`, and `phx.gen.json`; it defaults from the plural resource name. `phx.gen.context` can infer its context from the schema.

```console
$ mix phx.gen.live Post posts title:string
$ mix phx.new my_app --interactive
```

Use `--scope` when the generated resource belongs to a non-default boundary:

```console
$ mix phx.gen.live Blog Post posts title:string body:text --scope organization
```

New project generation may initialize a Git repository, provide interactive creation, and add project-level tooling. See [references/generators-client-and-testing.md](references/generators-client-and-testing.md) for the generated files and side effects.

## Assigns and channel-test helpers

`Phoenix.Socket.assign/2` accepts a function over existing assigns and merges the returned map. `Phoenix.Controller.assign/2` accepts the same functional form as well as maps and keyword lists.

```elixir
socket = Phoenix.Socket.assign(socket, fn assigns ->
  %{count: assigns.count + 1}
end)

conn = Phoenix.Controller.assign(conn, current_user: user, locale: "en")
```

Use guards directly with `assert_push`, `assert_broadcast`, and `assert_reply` when validating received payload shapes:

```elixir
assert_push "updated", payload when is_map(payload)
```

## JavaScript transport behavior

Expect the JavaScript socket to stop reconnect attempts while the page is hidden. LongPoll can use `fetch()` when `XMLHttpRequest` is unavailable, and Presence accepts a custom dispatcher for `presence_diff` broadcasts.

See the client reference for the related generator and testing behavior.
