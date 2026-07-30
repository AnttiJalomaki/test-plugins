---
name: shadcn-ui-knowledge-patch
description: shadcn/ui
version: 4.16.0
license: MIT
metadata:
  author: Nevaberry
---


# shadcn/ui Knowledge Patch

Use this skill when creating, upgrading, extending, or automating a shadcn/ui
project. Inspect the project before changing it: generated component source,
`components.json`, the package manifest, global CSS, and the selected primitive
base are the source of truth for an existing application.

## Reference Index

| Reference | Topics |
| --- | --- |
| [Upgrade and styling](references/upgrade-and-styling.md) | Tailwind v4, React 19 wrappers, CSS variables, animations, deprecations, palette refresh |
| [Projects, CLI, and presets](references/projects-cli-and-presets.md) | Initialization, templates, styles, presets, inspection, migrations, MCP, eject |
| [Registry consumption](references/registry-consumption.md) | Namespaces, authentication, discovery, dependency resolution, design-system and font items |
| [Registry authoring](references/registry-authoring.md) | Composition, validation, GitHub registries, loaders, targets, authentication errors |
| [Programmatic APIs](references/programmatic-apis.md) | Stable imports, configuration, caching, installation, typed errors, preset codes |
| [Component bases and content](references/component-bases-and-content.md) | Base UI, Radix, React Aria, Toast, chat helpers, Typeset, progressive migration |

## Start With Project Reality

Before running a write command:

1. Read `components.json` for aliases, style, registries, icon choice, and base.
2. Read the package manifest to identify React and Tailwind generations.
3. Inspect global CSS for Tailwind imports, `@theme`, color variables, and dark mode.
4. Open the installed component before assuming its primitive API or generated shape.
5. Commit local component changes before an overwrite, preset switch, or migration.

Use `info` for a machine-readable project summary and `docs` for the API matching
the selected component base:

```sh
pnpm dlx shadcn@latest info --json
pnpm dlx shadcn@latest docs combobox --base radix --json
```

## Breaking Changes and Deprecations

### Do not upgrade a project implicitly

Initialization supports Tailwind v4 and React 19, but adding a component to an
existing Tailwind v3 or React 18 project does not upgrade that project. Treat a
stack upgrade as separate work. Check Tailwind v4 browser support and run the
Tailwind upgrade codemod before adapting shadcn/ui CSS and components.

```sh
npx @tailwindcss/upgrade@next
```

### Replace deprecated defaults deliberately

- Prefer `sonner` over the legacy Radix-oriented `toast` component. Base UI has
  a distinct current Toast component; do not conflate the two.
- The old `default` style is deprecated and new projects use `new-york`.
  Existing projects retain their source rather than silently
  restyling installed components.
- Replace `tailwindcss-animate` and its `@plugin` directive with the
  `tw-animate-css` development dependency and a global CSS import.
- Buttons now keep the browser's default cursor. Add an explicit pointer cursor
  only when the product design requires it.

```css
@import "tw-animate-css";
```

### Treat `eject` as irreversible

New projects can import shared Tailwind v4 utilities from
`shadcn/tailwind.css`. `eject` copies those utilities into the project and
removes the `shadcn` dependency. The project will no longer receive later CLI
updates to that shared stylesheet automatically.

```sh
pnpm dlx shadcn@latest eject
```

## Tailwind v4 CSS Shape

Keep `:root` and `.dark` outside `@layer base`. Store a complete color value in
each custom property, then expose it through `@theme inline` without wrapping
the variable in another color function. Current palettes use OKLCH, and chart
configuration should also consume the complete variable directly.

```css
:root {
  --background: oklch(1 0 0);
  --chart-1: oklch(0.65 0.2 40);
}

.dark {
  --background: oklch(0.145 0 0);
}

@theme inline {
  --color-background: var(--background);
}
```

```ts
const chartConfig = { series: { color: "var(--chart-1)" } }
```

The refreshed dark palette is opt-in for applications upgraded from Tailwind
v3. Commit local changes, overwrite generated components, replace the dark
variables with the new OKLCH palette, and then reapply intentional
customizations.

```sh
pnpm dlx shadcn@latest add --all --overwrite
```

## React 19 Component Shape

Current wrappers can accept `React.ComponentProps` directly instead of using
`React.forwardRef`. Pass the ref through with the remaining props, put a
`data-slot` attribute on each primitive, and remove obsolete `displayName`
assignments.

```tsx
function AccordionItem({
  className,
  ...props
}: React.ComponentProps<typeof AccordionPrimitive.Item>) {
  return (
    <AccordionPrimitive.Item
      data-slot="accordion-item"
      className={cn("border-b last:border-b-0", className)}
      {...props}
    />
  )
}
```

## Choose and Preserve the Component Base

New projects default to Base UI. Scripts and CI that require Radix must request
it explicitly. `init --base` also accepts Aria where supported.

```sh
pnpm dlx shadcn@latest init --name dashboard --template astro --base radix
pnpm dlx shadcn@latest init -b radix
```

The public application imports remain the same across bases:

```tsx
import { Dialog, DialogContent, DialogTrigger } from "@/components/ui/dialog"
```

When a component is added, the CLI detects the configured base and transforms
both built-in and remote-registry source accordingly. Do not copy primitive
implementation details from a different base. Registries that need a specific
base should ship a `registry:base` item.

For customized Radix projects, migrate one component and its usages at a time
when practical. Radix and Base UI may coexist during the migration; typecheck,
build, and review behavioral differences after each component.

## Preview Before Writing

Use the `add` preflight flags to inspect registry effects:

```sh
pnpm dlx shadcn@latest add button --dry-run
pnpm dlx shadcn@latest add button --diff
pnpm dlx shadcn@latest add button --view
```

`--diff` is the safest starting point when an installed primitive contains
local changes. It compares local source with the registry update before a
merge. Use the standalone discovery commands when exploring remote content:

```sh
pnpm dlx shadcn view @acme/auth-system
pnpm dlx shadcn search @tweakcn -q "dark"
pnpm dlx shadcn list @acme
```

## Presets and Project Creation

A preset code contains colors, theme, icon library, fonts, and radius. Applying
one to an existing project may rewrite components, so inspect or commit first.
Limit a change to theme or font when component regeneration is unwanted.

```sh
pnpm dlx shadcn@latest preset decode a2r6bw --json
pnpm dlx shadcn@latest preset resolve --json
pnpm dlx shadcn@latest apply a2r6bw --only theme
```

`init` scaffolds supported frameworks, can create monorepos, and accepts a
preset. `create` is an alias. The interactive creator's style choice changes
component structure, spacing, fonts, and supporting libraries—not only colors.

## Registries

Define decentralized registries under a namespace in `components.json`. The
URL must contain `{name}`; object entries can also supply headers and parameters
with environment-variable interpolation.

```json
{
  "registries": {
    "@private": {
      "url": "https://registry.company.com/{name}.json",
      "headers": { "Authorization": "Bearer ${REGISTRY_TOKEN}" }
    }
  }
}
```

Dependencies are resolved before their dependent item. Configuration is
deep-merged, while the last resolved file wins when target paths collide. Use
that ordering intentionally for layered design systems, and pin every GitHub
dependency with its own tag or full commit SHA when reproducibility matters.

For authoring, validate source directly before building:

```sh
pnpm dlx shadcn registry validate
```

## Programmatic Use

Import only documented subpaths such as `shadcn/schema`, `shadcn/registry`, and
`shadcn/preset`; CLI command internals are not stable APIs. Direct API consumers
should use `getRegistry` instead of `fetchRegistry` and
`resolveRegistryItems` instead of `resolveRegistryTree`.

Registry reads cache resolved URLs for the life of the process and deduplicate
concurrent requests. Disable caching for servers or watchers that need fresh
content. `addRegistryItems` is non-interactive: pass fully resolved project
configuration, decide `overwrite` explicitly, and handle thrown typed registry
errors rather than expecting the process to exit.

Keep generated source reviewable. Use preflight output, preserve local edits,
and run the project's typecheck and build after component, preset, base, icon,
RTL, or registry-driven changes.
