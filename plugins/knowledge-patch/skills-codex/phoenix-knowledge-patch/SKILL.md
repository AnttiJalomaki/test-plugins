---
name: phoenix-knowledge-patch
description: Phoenix
version: 1.8.x
license: MIT
metadata:
  author: Nevaberry
---


# Phoenix Knowledge Patch

Use this skill when upgrading, generating, configuring, or debugging a Phoenix
application whose behavior may depend on recent framework and generator
changes. Check the application's Phoenix version and generated code before
applying advice: generators copy code into an application, so upgrading the
dependency does not rewrite previously generated files.

## Reference Index

| Reference | Topics |
| --- | --- |
| [references/upgrades-security-and-configuration.md](references/upgrades-security-and-configuration.md) | Runtime baseline, secure headers, endpoint configuration, deprecations, LongPoll hardening, logging, and verified routes |
| [references/scopes-and-authentication.md](references/scopes-and-authentication.md) | Generated scopes, scoped data access, route-aware scopes, magic links, sudo mode, and auth migration |
| [references/generators-layouts-and-projects.md](references/generators-layouts-and-projects.md) | Generator commands, layouts, themes, assets, new-project tooling, and generated side effects |
| [references/runtime-apis-and-javascript.md](references/runtime-apis-and-javascript.md) | Channel limits, assigns, channel assertions, socket behavior, LongPoll fallback, and Presence |

## Upgrade Hazards

### Verify the runtime first

Phoenix requires Erlang/OTP 25 or later. Check the deployed runtime as well as
the development environment before changing the dependency.

### Treat generated authentication as application code

`mix phx.gen.auth` output is not upgraded automatically. A pre-existing
password-registration flow remains in place until it is deliberately migrated.
Do not edit a historical migration to make `hashed_password` nullable; add a
new migration and plan how to handle unconfirmed accounts.

### Update controller declarations

Every `use Phoenix.Controller` must now provide `:formats`; use an empty list
when the controller has no formats.

```elixir
use Phoenix.Controller, formats: [:html]
```

Replace deprecated controller and router configuration:

- Remove `:namespace` and `:put_default_views` options.
- Name layout modules instead of using module-less layout tuples.
- Move trailing-slash URL generation from the router's deprecated
  `:trailing_slash` option to `Phoenix.VerifiedRoutes`.

```elixir
put_layout(conn, html: {MyAppWeb.Layouts, :print})
```

### Read endpoint compile-time settings explicitly

Code inside `Phoenix.Endpoint` no longer receives an injected `config`
variable. Read values needed during compilation with
`Application.compile_env/3`. A setting supplied only at runtime can otherwise
be missing when the endpoint module compiles.

```elixir
@value Application.compile_env(
  :my_app,
  [MyAppWeb.Endpoint, :some_setting],
  :default
)
```

### Review embedding policy

Without a caller-supplied Content Security Policy,
`put_secure_browser_headers` sets:

```text
base-uri 'self'; frame-ancestors 'self';
```

Applications embedded by another origin need an explicit policy. Do not rely
on `x-download-options` or `x-frame-options`; they are no longer added by this
function.

### Configure LongPoll deliberately

LongPoll is opt-in. If it is enabled, keep the body-splitting security fixes and
the 100-event request limit in mind. High-frequency clients that can exceed the
limit need an explicit upgrade sequence; see the security reference before
deploying that path.

## Generated Scopes: The New Data-Access Shape

### Pass scope into contexts

Generated authentication creates an `Accounts.Scope` struct and normally
registers a default user scope. Generated controllers and LiveViews pass
`:current_scope` as the first argument to context operations.

```elixir
def list_posts(%Scope{} = scope) do
  Repo.all(from post in Post, where: post.user_id == ^scope.user.id)
end
```

Once a default scope is configured, schema, context, LiveView, HTML, and JSON
generators emit ownership fields and scoped queries. Preserve that boundary in
hand-written context functions; do not fall back to unqualified queries.

### Mount scope before authenticated LiveViews

Place generated authenticated LiveView routes in the authenticated
`live_session`. Its mount hook establishes the scope expected by generated
operations.

### Configure scope generation, not just runtime assigns

Scope definitions live in application configuration. The definition connects
the runtime scope module and assign with generated schema ownership, fixtures,
tests, and routes. There can be only one default scope, but additional named
scopes can be selected with `--scope`.

For nested route scopes, use `route_prefix` and `route_access_path` for the
URL-facing identity while retaining `access_path` for the ownership key. Load a
nested owner through the existing user scope, then replace `:current_scope` in
both the browser plug and LiveView `on_mount` hook.

## Authentication Quick Reference

The current auth generator is magic-link-first:

- Registration no longer asks for a password.
- Email confirmation and magic-link login are generated by default.
- Password authentication is opt-in.
- `require_sudo_mode` protects operations that need recent authentication.
- `require_authenticated_user` must run after
  `fetch_current_scope_for_user`.

When migrating older generated auth, make `hashed_password` nullable in a new
migration and clear it for still-unconfirmed accounts to prevent credential
pre-stuffing. Because this can invalidate a password just chosen by a new user,
deploy at low traffic or add magic links without immediately replacing the
existing flow.

## Layout and Generator Quick Reference

### Invoke dynamic layouts as components

New applications use one `root.html.heex` around the render pipeline. Invoke a
dynamic layout such as the application layout from HEEx as a function
component.

```heex
<Layouts.app flash={@flash}>
  ...
</Layouts.app>
```

### Use shortened generator forms

The context argument is optional for `phx.gen.live`, `phx.gen.html`, and
`phx.gen.json`; it defaults from the plural resource name.
`phx.gen.context` can infer a context from the schema.

```console
$ mix phx.gen.live Post posts title:string
$ mix phx.new my_app --interactive
```

If `phx.gen.auth` reports that esbuild is unavailable, ensure the JavaScript
bundle includes `phoenix_html.js`; generated auth features assume it is loaded.

## Runtime API Quick Reference

### Bound channel fan-out

`max_channels_per_transport` defaults to 100. Raise it explicitly only when a
client is expected and permitted to multiplex more than 100 channel processes
over one transport.

### Assign values in bulk

`Phoenix.Socket.assign/2` accepts a function of existing assigns and merges the
map it returns. `Phoenix.Controller.assign/2` supports the same functional form
as well as maps and keyword lists.

```elixir
socket = Phoenix.Socket.assign(socket, fn assigns ->
  %{count: assigns.count + 1}
end)

conn = Phoenix.Controller.assign(conn, current_user: user, locale: "en")
```

### Guard channel assertions

`assert_push`, `assert_broadcast`, and `assert_reply` accept guards, so payload
shape can be constrained in the receive assertion.

```elixir
assert_push "updated", payload when is_map(payload)
```

## Verification Checklist

Before finishing a Phoenix change:

1. Confirm the Phoenix and Erlang/OTP versions in the project and deployment.
2. Separate dependency behavior from code copied by older generators.
3. Search for `use Phoenix.Controller`, endpoint `config` access, legacy layout
   tuples, and router `:trailing_slash` configuration.
4. Verify authenticated LiveViews run inside the correct `live_session`.
5. Exercise scoped context functions with the intended owner and nested route
   scope.
6. Test embedding headers, LongPoll activation, transport fan-out, and verified
   route generation where those paths are used.
7. Inspect generated-project side effects before accepting a generator diff.

