# Generators, JavaScript Clients, and Channel Tests

This task-oriented reference includes generator, client, and testing behavior from the `1.8.x` batch, plus the generated-auth asset requirement associated with current authentication guidance.

## Streamlined resource generators

The context argument is optional for:

- `phx.gen.live`
- `phx.gen.html`
- `phx.gen.json`

When omitted, the generator derives the context from the plural resource name. `phx.gen.context` can likewise infer a context from the schema.

For example:

```console
$ mix phx.gen.live Post posts title:string
```

This short form derives the context rather than requiring it as a separate positional argument.

When a resource belongs to a configured non-default generator scope, keep the explicit `--scope` selection:

```console
$ mix phx.gen.live Blog Post posts title:string body:text --scope organization
```

The scope changes generated ownership fields, context queries, routes, fixtures, and setup integration. See `scopes-and-auth.md` for the full boundary configuration.

## Interactive project generation

`phx.new` supports an interactive mode:

```console
$ mix phx.new my_app --interactive
```

Use it when choices should be gathered during generation.

## Generated project side effects and tooling

When Git is installed, `phx.new` initializes a repository.

The `--docker` option uses Debian trixie as its base image.

Generated projects include:

- A `mix precommit` alias.
- An `AGENTS.md` compatible with `usage_rules`.
- A `usage_rules` directory for synchronizing Phoenix guidance.

## Tailwind-enabled application themes

Tailwind-enabled generated applications use daisyUI-backed themes with light, dark, and system choices.

## Authentication generator assets

The features emitted by `phx.gen.auth` assume that `phoenix_html.js` is included in the JavaScript bundle. The generator warns when esbuild is unavailable.

## JavaScript socket visibility behavior

The JavaScript socket stops reconnection attempts while the page is hidden.

## LongPoll fallback transport

LongPoll can use `fetch()` when `XMLHttpRequest` is unavailable.

LongPoll remains opt-in after Phoenix 1.8.2. Its server-side security and event-batch constraints are detailed in `runtime-security-and-routing.md`.

## Presence dispatch and key safety

Presence supports a custom dispatcher for `presence_diff` broadcasts.

Phoenix 1.8.9 also prevents Presence keys that match `Object.prototype` members from crashing the JavaScript client.

## Guarded channel assertions

As of Phoenix 1.8.4, `assert_push`, `assert_broadcast`, and `assert_reply` accept guards. Constrain a received payload inline instead of adding a separate assertion solely for its basic shape.

```elixir
assert_push "updated", payload when is_map(payload)
```

The same pattern applies to broadcast and reply assertions.

## Deterministic verified-route queries in tests

The top-level Phoenix setting added in 1.8.3 makes verified-route query-parameter ordering deterministic:

```elixir
config :phoenix, sort_verified_routes_query_params: true
```

Tests can enable it when they need deterministic generated query strings.
