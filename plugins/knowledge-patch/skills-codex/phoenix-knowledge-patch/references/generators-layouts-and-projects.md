# Generators, Layouts, and Generated Projects

This topic reference incorporates generator and project behavior from batch
`1.8.x`.

## Render Layouts as Function Components

New applications keep a single `root.html.heex` around the render pipeline.
Dynamic layouts such as `app.html.heex` are regular function components invoked
from templates, not extra layouts configured in the render pipeline.

```heex
<Layouts.app flash={@flash}>
  ...
</Layouts.app>
```

When updating older code, pair this component model with module-qualified
layout calls. Module-less layouts are deprecated.

## Use Streamlined Generator Commands

The context argument to `phx.gen.live`, `phx.gen.html`, and `phx.gen.json` is
optional. When omitted, the generator derives the context from the plural
resource name. `phx.gen.context` can similarly infer the context from the
schema.

```console
$ mix phx.gen.live Post posts title:string
```

`phx.new` also offers an interactive mode:

```console
$ mix phx.new my_app --interactive
```

Scope-aware generation is covered separately in
[scopes-and-authentication.md](scopes-and-authentication.md). Once a default
scope exists, schema and web generators produce ownership fields and scoped
queries; use `--scope` to select a non-default scope.

## Include the Authentication JavaScript Asset

`phx.gen.auth` warns when esbuild is unavailable. Its generated functionality
assumes that `phoenix_html.js` is included in the JavaScript bundle, so confirm
the asset is loaded when using a different bundler or asset setup.

## Generated Application Defaults

Tailwind-enabled generated applications use daisyUI-backed light, dark, and
system themes. The generated development setup:

- Honors the `PORT` environment variable.
- Enables HEEx `:debug_tags_location`.

Generated `prod.exs` enables `force_ssl` by default. Review this setting
against the deployment's proxy and TLS termination rather than assuming an
older generated configuration.

## New-Project Tooling and Side Effects

Review the whole generated tree before accepting a new-project diff:

- When Git is installed, `phx.new` initializes a repository.
- `phx.new --docker` uses Debian trixie as its base image.
- Generated projects include a `mix precommit` alias.
- Generated projects include an `AGENTS.md` compatible with `usage_rules`.
- A generated `usage_rules` directory supports synchronizing Phoenix
  guidance.

Repository initialization is a generator side effect, not merely a generated
file. Account for it when invoking `phx.new` inside another repository or an
automated build directory.

