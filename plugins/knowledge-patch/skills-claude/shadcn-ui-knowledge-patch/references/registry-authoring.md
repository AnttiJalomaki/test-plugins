# Registry Authoring

## Compose Source Registries

A root `registry.json` may collect items from other registry files with
`include`. Only the root must define `name` and `homepage`:

```json
{
  "$schema": "https://ui.shadcn.com/schema/registry.json",
  "name": "acme",
  "homepage": "https://acme.com",
  "include": [
    "components/ui/registry.json",
    "hooks/registry.json"
  ]
}
```

`shadcn build` resolves those includes to a flat generated registry, removes
the `include` field, and preserves each item file path relative to the root.

## Validate From Source

Validation does not require a build first:

```sh
pnpm dlx shadcn registry validate
```

It checks the root and included registries, item schemas, duplicate item names,
include constraints, and local file paths. It reports all actionable errors in
one run so authors can fix the complete set together.

## Load Dynamic Registry Routes

Use the documented `shadcn/registry` entry point to load the composed registry
or one resolved item in a dynamic route:

```ts
import { loadRegistry, loadRegistryItem } from "shadcn/registry"

const registry = await loadRegistry()
const item = await loadRegistryItem(name)
```

Do not import CLI command internals for this purpose.

## Distribute From a Public GitHub Repository

Any public repository with `registry.json` at its root can be consumed as
`<username>/<repo>/<item>`:

```sh
pnpm dlx shadcn@latest add acme/toolkit/project-conventions
```

The CLI reads the source registry and resolves its includes. The author does
not need to run `shadcn build`, host generated item JSON, or operate a registry
server. A `registry:item` may distribute arbitrary project files, including
documentation, editor settings, agent instructions, workflows, templates, and
codemods.

## Return Useful Authentication Errors

An authenticated registry may respond to a `401` or `403` with JSON containing
a `message`. The CLI shows that message to the user, allowing a registry to
explain a missing token, expired subscription, or resource-specific permission.

```ts
return NextResponse.json(
  {
    error: "Forbidden",
    message: "This component requires Design team access.",
  },
  { status: 403 }
)
```

Keep the response safe for display and do not include secrets.

## Target Files Through Consumer Aliases

A registry file target may begin with `@components/`, `@ui/`, `@lib/`, or
`@hooks/`. The CLI resolves it against the consumer's `components.json`
directories, independently of the consumer's import prefix.

`@utils/` is not supported because that alias identifies a file rather than a
directory. A target may intentionally route a file somewhere different from
its declared type. `registry:page` and `registry:file` entries require a target.

```json
{
  "path": "registry/new-york/example/format-date.ts",
  "type": "registry:ui",
  "target": "@lib/format-date.ts"
}
```
