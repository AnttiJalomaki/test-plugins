# Registries and Programmatic APIs

## Configure namespaced registries

CLI 3.0 addresses decentralized registry items as `@registry/name`. Configure a
namespace in `components.json` with a URL template containing `{name}`:

```json
{
  "registries": {
    "@acme": "https://acme.com/r/{name}.json"
  }
}
```

```sh
pnpm dlx shadcn add @acme/button
```

An item can declare namespaced `registryDependencies` from one or several
registries. The CLI resolves all of them automatically. This namespaced workflow
comes from `cli-3-and-mcp`.

## Follow namespace and URL-template constraints

A namespace must start and end with an alphanumeric character. Between those
ends it may contain alphanumerics, hyphens, or underscores. A registry URL must
contain `{name}` and may contain `{style}` for the project's current style.

Object-form entries can add `params`. Environment expansion works in the URL,
headers, and parameters, including shell-style defaults such as
`${REGISTRY_VERSION:-v2}`:

```json
{
  "registries": {
    "@themes_v2": {
      "url": "https://registry.example.com/{style}/{name}.json",
      "params": {
        "version": "${REGISTRY_VERSION:-v2}"
      }
    }
  }
}
```

These constraints come from `registry-configuration`.

## Authenticate private registries

A registry entry can be an object with `url` and `headers`. CLI 3.0 supports
basic authentication, bearer tokens, API-key query parameters, and custom
headers. Values can interpolate environment variables:

```json
{
  "registries": {
    "@private": {
      "url": "https://registry.company.com/{name}.json",
      "headers": {
        "Authorization": "Bearer ${REGISTRY_TOKEN}"
      }
    }
  }
}
```

When a variable is missing, the CLI names it in the error. Supply variables via
`.env` or `.env.local`; do not commit their secret values. Private-registry
authentication comes from `cli-3-and-mcp`.

An authenticated registry can return a JSON `message` with a `401` or `403`.
The CLI displays the message, allowing the service to explain a missing token,
expired subscription, or resource-specific access restriction:

```ts
return NextResponse.json(
  { error: "Forbidden", message: "This component requires Design team access." },
  { status: 403 }
)
```

This response behavior comes from `registry-configuration`.

## Compose source registries

A root `registry.json` can include other registry files. Only the root needs
`name` and `homepage`:

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

`shadcn build` resolves all includes into a flat registry with no `include`
field and preserves item file paths relative to the root.

Validate the source before building:

```sh
pnpm dlx shadcn registry validate
```

Validation covers the root, included registries, item schemas, duplicate item
names, include rules, and local file paths. It reports all actionable errors in
one run and does not require a prior build. Composition and validation come from
`registry-composition-and-github`.

## Load source registries dynamically

Dynamic registry routes can load the composed registry or one resolved item
from the documented `shadcn/registry` entry point:

```ts
import { loadRegistry, loadRegistryItem } from "shadcn/registry"

const registry = await loadRegistry()
const item = await loadRegistryItem(name)
```

## Distribute items from GitHub

Any public GitHub repository with a root `registry.json` can be addressed as
`<username>/<repo>/<item>`:

```sh
pnpm dlx shadcn@latest add acme/toolkit/project-conventions
```

The CLI reads the source registry and resolves its includes. The author does not
need to run `shadcn build`, publish generated item JSON, or host a registry
server. A `registry:item` can carry arbitrary project files, including
documentation, editor settings, agent instructions, workflows, templates, and
codemods. GitHub source registries come from
`registry-composition-and-github`.

## Control dependency ordering and overrides

Registry dependencies install before the item that declares them. Resolution
deep-merges Tailwind settings, CSS variables, CSS, environment variables, and
other configuration. When resolved items target the same file path, the last
resolved file wins. This permits a custom item to depend on a third-party item
and override only selected files or settings:

```json
{
  "name": "custom-button",
  "type": "registry:ui",
  "registryDependencies": ["@vendor/button"],
  "files": [
    {
      "path": "components/ui/button.tsx",
      "type": "registry:ui",
      "content": "export function Button() { return null }"
    }
  ]
}
```

Use duplicate target paths only as intentional overrides. This resolution model
comes from `registry-configuration`.

## Publish design-system and font items

A `registry:base` item can carry a complete design system: components,
dependencies, CSS variables, fonts, and configuration. It also pins the desired
primitive base for initialization.

Fonts are separately installable `registry:font` items. Their metadata includes
the provider, import name, family, CSS variable, and subsets:

```json
{
  "$schema": "https://ui.shadcn.com/schema/registry-item.json",
  "name": "font-inter",
  "type": "registry:font",
  "font": {
    "family": "'Inter Variable', sans-serif",
    "provider": "google",
    "import": "Inter",
    "variable": "--font-sans",
    "subsets": ["latin"]
  }
}
```

These item types come from `cli-v4-and-presets`.

## Use documented programmatic entry points

Only documented subpath imports are stable. CLI command internals are not a
public API. For CLI 3.0 consumers:

- replace `fetchRegistry` with `getRegistry`;
- replace `resolveRegistryTree` with `resolveRegistryItems`; and
- import registry schemas from `shadcn/schema`.

```ts
import { registryItemSchema } from "shadcn/schema"
```

Existing `components.json` files and installed components remain compatible.
The API migration itself comes from `cli-3-and-mcp`; the more detailed API
behavior below comes from `registry-schema-and-api`.

## Resolve configuration and control caching

Registry fetches cache by resolved URL for the process lifetime and deduplicate
concurrent in-flight requests. Disable the cache for fresh reads in servers and
watchers:

```ts
const config = await getRegistriesConfig(process.cwd())
const items = await getRegistryItems(["@acme/button"], {
  config,
  useCache: false,
})
```

`getRegistriesConfig(cwd)` reads `components.json`. If that is absent, it falls
back to the top-level `registries` property in `package.json`.

## Install registry items without prompts

`addRegistryItems` writes files and applies dependencies, environment variables,
CSS, and Tailwind configuration without prompting. It throws instead of exiting
and skips existing files unless `overwrite` is true.

The function does not load project configuration. Pass a resolved configuration
with aliases and `resolvedPaths`:

```ts
const cwd = process.cwd()
const config = await getRegistriesConfig(cwd)
await addRegistryItems(["@acme/agent"], {
  cwd,
  config,
  overwrite: false,
  silent: true,
})
```

A registries-only configuration is sufficient only for universal
`registry:item` or `registry:file` payloads whose files all declare explicit
targets.

## Handle typed registry failures

Registry functions throw `RegistryError` subclasses rather than terminating the
process. Branch on `RegistryErrorCode` or catch the specific classes for:

- missing registries or items;
- authentication;
- network fetches;
- configuration;
- local files;
- parsing and validation;
- invalid namespaces; and
- missing environment variables.

```ts
try {
  await getRegistry("@unknown")
} catch (error) {
  if (error instanceof RegistryNotFoundError) {
    // Recover from an unknown registry.
  }
}
```

## Pin GitHub and local dependencies

`registryDependencies` may reference GitHub items or local item JSON files. A
GitHub dependency does not inherit a ref from its parent, so give every GitHub
dependency its own tag or full commit SHA for reproducibility. Bare names still
resolve to built-in items; a dependency from the same GitHub repository must use
its full GitHub address.

```json
{
  "registryDependencies": [
    "acme/ui/button#v1.2.0",
    "./editor.json"
  ]
}
```

## Route files with consumer aliases

A file target can start with `@components/`, `@ui/`, `@lib/`, or `@hooks/`.
These resolve against the corresponding directories in the consumer's
`components.json`, independently of the import prefix. `@utils/` is not
supported because that alias identifies a file rather than a directory.

The target may send a file somewhere different from the location implied by its
declared type. A target is required for `registry:page` and `registry:file`.

```json
{
  "path": "registry/new-york/example/format-date.ts",
  "type": "registry:ui",
  "target": "@lib/format-date.ts"
}
```

## Encode and decode preset codes

`encodePreset` accepts a partial preset, fills missing fields from
`DEFAULT_PRESET_CONFIG`, and returns a version-prefixed, URL-safe code.
`decodePreset` returns the complete defaulted configuration, or `null` when the
code is absent or invalid.

```ts
import { decodePreset, encodePreset } from "shadcn/preset"

const code = encodePreset({ style: "vega", theme: "blue", radius: "large" })
const preset = decodePreset(code)
```

`shadcn/preset` also exports validators, random-preset helpers, Base62 helpers,
and the `PRESET_*` option constants used by theme tooling.
