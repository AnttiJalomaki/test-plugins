# Registry Consumption

## Configure Namespaced Registries

Address decentralized registry items as `@namespace/name`. Define the
namespace in `components.json` with a URL template that contains `{name}`:

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

A namespace must start and end with an alphanumeric character. Between those
characters it may contain alphanumerics, hyphens, or underscores. The template
may also use `{style}`, which expands to the project's current style.

Object-form entries accept `url`, `headers`, and `params`. Environment
variables expand in all three locations, and shell-style defaults are allowed:

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

## Authenticate Private Registries

Registry configuration supports basic authentication, bearer tokens, API-key
query parameters, and arbitrary request headers. Keep secrets in environment
variables rather than committing them:

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

When a required variable is absent, the CLI identifies it by name. Supply it
through the process environment, `.env`, or `.env.local` as appropriate for the
project.

## Discover Before Installing

Use `view` to inspect an item, `search` to query a registry, and `list` to
enumerate it:

```sh
pnpm dlx shadcn view @acme/auth-system
pnpm dlx shadcn search @tweakcn -q "dark"
pnpm dlx shadcn list @acme
```

For an `add` operation, `--dry-run`, `--view`, and `--diff` provide more
project-specific preflight information before any file is written.

## Resolve Dependencies and Overrides

An item can use namespaced `registryDependencies`, including dependencies from
several registries. The CLI resolves and installs dependencies before the item
that requested them.

Resolution deep-merges configuration including Tailwind settings, CSS
variables, CSS, and environment variables. If multiple resolved files target
the same path, the last resolved file wins. This permits an intentional layer:
a custom item can depend on a third-party item and replace only chosen files or
configuration.

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

Dependencies may also point to public GitHub items or local item JSON files:

```json
{
  "registryDependencies": [
    "acme/ui/button#v1.2.0",
    "./editor.json"
  ]
}
```

GitHub refs are not inherited. Give every GitHub dependency its own tag or full
commit SHA for reproducibility. A bare name resolves to a built-in item, so a
same-repository dependency still needs its full GitHub address.

## Install Complete Design Systems and Fonts

A `registry:base` item can install a complete design-system payload:
components, dependencies, CSS variables, fonts, and configuration. It also lets
registry authors pin the intended primitive base.

Fonts are independently installable `registry:font` items. Their metadata can
declare the family, provider, import name, CSS variable, and subsets:

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
