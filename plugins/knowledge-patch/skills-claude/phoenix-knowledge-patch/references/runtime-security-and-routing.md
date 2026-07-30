# Runtime, Security, Routing, and Realtime Behavior

This reference groups runtime and server behavior from the `1.8.x` batch by operational concern.

## Runtime and generated application defaults

Phoenix 1.8 requires Erlang/OTP 25 or later.

Tailwind-enabled generated applications use daisyUI-backed light, dark, and system themes. Their development setup:

- Honors the `PORT` environment variable.
- Enables HEEx `:debug_tags_location`.

Generated `prod.exs` enables `force_ssl` by default.

## Layout rendering

New applications have a single `root.html.heex` surrounding the render pipeline. Dynamic layouts such as `app.html.heex` are ordinary function components invoked from templates rather than extra layouts configured in the render pipeline.

```heex
<Layouts.app flash={@flash}>
  ...
</Layouts.app>
```

Module-less layouts are deprecated. When setting a layout from a controller, identify its module:

```elixir
put_layout(conn, html: {MyAppWeb.Layouts, :print})
```

## Secure browser headers

When the caller supplies no Content Security Policy, `put_secure_browser_headers` now emits:

```text
content-security-policy: base-uri 'self'; frame-ancestors 'self';
```

The `frame-ancestors 'self'` directive blocks third-party embedding. Applications intentionally embedded by another origin need an explicit policy.

The helper no longer sets the deprecated `x-download-options` or `x-frame-options` headers. Do not write tests or downstream policy logic that assumes those defaults remain present.

## Controller and router migration

`use Phoenix.Controller` must specify `:formats`; an empty list is valid when appropriate.

```elixir
use Phoenix.Controller, formats: [:html]
```

The following forms are deprecated:

- The controller's `:namespace` option.
- The controller's `:put_default_views` option.
- Module-less layouts.
- The router's `:trailing_slash` option.

Move trailing-slash URL generation to `Phoenix.VerifiedRoutes`.

## Endpoint compile-time configuration

The former injected `config` variable is unavailable inside `Phoenix.Endpoint`. Read compile-time settings explicitly with `Application.compile_env/3`.

```elixir
@value Application.compile_env(
  :my_app,
  [MyAppWeb.Endpoint, :some_setting],
  :default
)
```

An upgrade may expose settings that were supplied only at runtime even though endpoint code consumes them at compile time. Audit those settings because this mismatch may result in boot errors.

## Verified routes

Since Phoenix 1.8.6, a module raises if it invokes `use Phoenix.VerifiedRoutes` more than once. It also raises if a list is interpolated into a verified route.

Tests may request deterministic query-parameter ordering with the top-level Phoenix setting added in 1.8.3:

```elixir
config :phoenix, sort_verified_routes_query_params: true
```

## Bulk and functional assigns

`Phoenix.Socket.assign/2` accepts a function of the current assigns. The map returned by the function is merged into the socket assigns.

```elixir
socket = Phoenix.Socket.assign(socket, fn assigns ->
  %{count: assigns.count + 1}
end)
```

`Phoenix.Controller.assign/2` supports the same functional form and also accepts maps and keyword lists, aligning controller assignment with LiveView-style bulk assignment.

```elixir
conn = Phoenix.Controller.assign(conn, current_user: user, locale: "en")
```

## Channel process limits

Phoenix 1.8.9 adds `max_channels_per_transport`, defaulting to 100. This bounds the number of channel processes one client can create over a transport.

If an application intentionally multiplexes more than 100 channels over a single transport, raise the option explicitly.

## LongPoll activation and hardening

LongPoll has several patch-level changes that must be considered together:

- Phoenix 1.8.0 inadvertently enabled LongPoll by default.
- Phoenix 1.8.2 restored it to opt-in.
- Phoenix 1.8.6 fixed memory exhaustion caused by nd-JSON body splitting.
- Phoenix 1.8.9 enforces the resulting 100-event request batch limit.

A high-frequency LongPoll application that may exceed 100 events in one request should upgrade to Phoenix 1.8.7 before moving to 1.8.9.

The JavaScript LongPoll client can use `fetch()` when `XMLHttpRequest` is unavailable.

## Presence and log hardening

Phoenix 1.8.9 prevents JavaScript Presence keys matching members of `Object.prototype` from crashing the client.

Since Phoenix 1.8.7, the default log filter masks a `token` parameter alongside `password`.
