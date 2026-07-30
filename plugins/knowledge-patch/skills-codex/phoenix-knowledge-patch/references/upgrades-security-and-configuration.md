# Upgrades, Security, and Configuration

This topic reference incorporates upgrade and runtime changes from batch
`1.8.x`.

## Runtime Baseline

Phoenix 1.8 requires Erlang/OTP 25 or later. Validate the deployed Erlang/OTP
version before upgrading the Phoenix dependency, including the runtime inside
the release image or host rather than only the local toolchain.

## Secure Browser Headers

When the caller does not supply a Content Security Policy,
`put_secure_browser_headers` now sets `content-security-policy` to:

```text
base-uri 'self'; frame-ancestors 'self';
```

The `frame-ancestors 'self'` directive blocks third-party framing. Provide an
explicit policy when cross-origin embedding is an intended part of the
application.

The function no longer adds the deprecated `x-download-options` or
`x-frame-options` headers. If downstream checks still require either header,
decide whether to update the check or add an explicit application policy; do
not assume Phoenix supplies it.

## Controller and Router Deprecations

`use Phoenix.Controller` must specify `:formats`; an empty list is valid.

```elixir
use Phoenix.Controller, formats: [:html]
```

The following forms are deprecated:

- The controller `:namespace` option.
- The controller `:put_default_views` option.
- Layouts without a module name.
- The router's `:trailing_slash` option.

Name the layout module explicitly:

```elixir
put_layout(conn, html: {MyAppWeb.Layouts, :print})
```

Move trailing-slash URL generation to `Phoenix.VerifiedRoutes`.

## Endpoint Compile-Time Configuration

The `config` variable is no longer injected inside `Phoenix.Endpoint`. Read a
compile-time value with `Application.compile_env/3`:

```elixir
@value Application.compile_env(
  :my_app,
  [MyAppWeb.Endpoint, :some_setting],
  :default
)
```

Audit whether each value is present at compile time. Configuration previously
provided only at runtime can cause a boot failure after the endpoint has been
compiled without it.

## LongPoll Activation and Hardening

Phoenix 1.8.0 inadvertently enabled LongPoll by default; Phoenix 1.8.2 restored
it to opt-in. Configure it explicitly rather than relying on the brief default
behavior.

The security and request-limit sequence matters:

- Phoenix 1.8.6 fixes memory exhaustion caused by nd-JSON body splitting.
- Phoenix 1.8.9 enforces the resulting limit of 100 events per batch.
- A high-frequency LongPoll application that may exceed 100 events in one
  request should first move to Phoenix 1.8.7, then adapt before moving to
  Phoenix 1.8.9.

Keep LongPoll opt-in, the body-splitting fix, and the batch limit separate when
diagnosing an upgrade: they were introduced in different patch releases.

## Presence and Log Hardening

Phoenix 1.8.9 prevents JavaScript Presence keys that match
`Object.prototype` members from crashing the client. Phoenix masks a `token`
parameter in logs by default since 1.8.7, alongside `password`. If an
application previously depended on logging a parameter named `token`, rename
or instrument that value deliberately instead of weakening the default mask.

## Stricter Verified Routes

Phoenix 1.8.6 raises in two cases that previously escaped validation:

- A module invokes `use Phoenix.VerifiedRoutes` more than once.
- A list is interpolated into a verified route.

Remove duplicate `use` declarations and serialize repeated query values in a
supported form rather than interpolating a list directly.

Since Phoenix 1.8.3, tests can request deterministic ordering of verified-route
query parameters with a top-level Phoenix setting:

```elixir
config :phoenix, sort_verified_routes_query_params: true
```

Use the setting where assertions compare complete URLs and ordering would
otherwise make the result nondeterministic.

