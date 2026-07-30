---
name: shadcn-ui-knowledge-patch
description: shadcn/ui
version: 4.16.0
license: MIT
metadata:
  author: Nevaberry
---


# shadcn/ui Knowledge Patch

Use this skill when creating, upgrading, inspecting, or automating a shadcn/ui
project. It covers current CLI behavior, component bases, Tailwind v4 and React
19 transitions, presets, registries, and the supported programmatic APIs.

## Start with the project

Before suggesting commands or editing copied component source:

1. Read `components.json`, the package manifest, and the global stylesheet.
2. Run `shadcn info --json` when the CLI is available to identify the framework,
   component base, CSS-variable setup, installed components, and resources.
3. Preserve the project's existing stack. Adding a component does not silently
   move a Tailwind v3 or React 18 app to Tailwind v4 or React 19.
4. Determine whether the project uses Base UI, Radix, or React Aria. Do not apply
   primitive-specific APIs or registry output across bases without conversion.
5. Inspect local component modifications before overwriting registry files.

## Reference index

| Reference | Topics |
| --- | --- |
| [Project setup and styling](references/project-setup-and-styling.md) | Tailwind v4, React 19, CSS variables, palettes, animation CSS, scaffolding, styles, presets, and shared utilities |
| [Components and bases](references/components-and-bases.md) | Base UI, Radix, React Aria, toast behavior, progressive migration, deterministic chat helpers, and Typeset |
| [CLI and migrations](references/cli-and-migrations.md) | Discovery, preflight inspection, project context, MCP, built-in migrations, and CLI skill installation |
| [Registries and programmatic APIs](references/registries-and-programmatic-apis.md) | Namespaces, authentication, composition, GitHub sources, schemas, file targets, caching, installation, errors, and preset APIs |

## Breaking changes and deprecations

### Keep upgrades explicit

- New initialization can produce Tailwind v4 and React 19 projects.
- Existing Tailwind v3 and React 18 projects remain on their current stack,
  including when components are added.
- Before a Tailwind v4 upgrade, check its browser-compatibility boundary and run
  the `@tailwindcss/upgrade@next` codemod.
- The March 2025 dark palette is automatic for projects created on Tailwind v4,
  but not for projects upgraded from v3. Apply that palette as a separate,
  opt-in overwrite-and-review operation.

### Update Tailwind v4 tokens correctly

Place `:root` and `.dark` outside `@layer base`. Store a complete color value in
each CSS variable and map it directly through `@theme inline`:

```css
:root {
  --background: hsl(0 0% 100%);
}

.dark {
  --background: hsl(0 0% 3.9%);
}

@theme inline {
  --color-background: var(--background);
}
```

Do not wrap the variable in another color function. New palettes use OKLCH.
Chart configuration likewise consumes the complete token, for example
`color: "var(--chart-1)"`.

### Modernize React wrappers without changing behavior

React 19 component source uses functions typed with `React.ComponentProps`,
passes refs with the remaining props, and marks each primitive with `data-slot`.
Remove obsolete `React.forwardRef` wrappers and their `displayName` assignments.

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

### Distinguish old defaults from base-specific additions

- The legacy `toast` component is deprecated in favor of `sonner`.
- Base UI projects also have a newer Base UI Toast with actions, statuses,
  promises, stacking, and swipe dismissal. Do not confuse it with the deprecated
  legacy implementation.
- The `default` style is deprecated; newer projects use `new-york`.
- Buttons retain the browser's default cursor.
- `tailwindcss-animate` is deprecated. Remove its dependency and `@plugin`
  directive, add `tw-animate-css` as a development dependency, and import
  `tw-animate-css` from global CSS.

### Treat ejection as a one-way ownership change

New initialization imports `shadcn/tailwind.css` for shared Tailwind v4
variants, utilities, and animations. `shadcn eject` inlines that stylesheet and
removes the `shadcn` dependency. After ejection, later CLI stylesheet updates no
longer flow into the project automatically.

## Current component-base rules

- Base UI is the default for new projects. Automation that requires Radix must
  pass `--base radix` or `-b radix` explicitly.
- A registry that must pin its primitive base should ship a `registry:base`
  item; otherwise initialization now selects Base UI.
- Base UI preserves the shadcn/ui component abstraction, imports, appearance,
  and behavior. The CLI detects the selected library and transforms built-in
  and remote-registry components for it.
- React Aria is isolated in its own registry. It supports Vega, Nova, Maia,
  Lyra, Mira, Luma, Rhea, and Sera without changing existing Base UI or Radix
  components.
- Customized Radix projects can migrate incrementally. Keep Radix and Base UI
  side by side while migrating one component and its usages at a time.

## Safe CLI workflow

Use inspection before mutation:

```sh
pnpm dlx shadcn@latest info --json
pnpm dlx shadcn@latest add button --view
pnpm dlx shadcn@latest add button --dry-run
pnpm dlx shadcn@latest add button --diff
```

- `--view` shows the registry item, `--dry-run` previews planned changes, and
  `--diff` compares an installed primitive with its registry update.
- Use `docs <component> --base <base> --json` for component documentation,
  examples, and primitive API details that match the selected base.
- Use `view`, `search`, and `list` to inspect decentralized registries before
  installation.
- Initialize the project MCP server with `shadcn@latest mcp init`; it uses all
  registries configured for the project.

## Scaffolding and presets

`init` scaffolds supported frameworks and `create` is its alias. Select a
component base explicitly when reproducibility matters:

```sh
pnpm dlx shadcn@latest init --name dashboard --template astro --base radix
pnpm dlx shadcn@latest init --template next --monorepo
```

A preset code bundles colors, theme, icon library, fonts, and radius. It can
configure a new app or reconfigure an existing one, including component source.
Inspect codes before applying them and use `--only theme` or `--only font` when
the component set should remain untouched.

```sh
pnpm dlx shadcn@latest preset decode a2r6bw --json
pnpm dlx shadcn@latest preset resolve --json
pnpm dlx shadcn@latest apply a2r6bw --only theme
```

## Registry essentials

- Configure decentralized registries under names such as `@acme` with a URL
  template containing `{name}`. Namespace syntax is constrained; consult the
  registry reference before generating a name.
- Object-form registry entries can carry headers and parameters with environment
  interpolation. Never hard-code private tokens in `components.json`.
- Registry dependencies install first. Configuration is deep-merged, while a
  duplicate target file path is won by the last resolved file; use this only for
  intentional overrides.
- A public GitHub repository with a root `registry.json` can be consumed directly
  as `<username>/<repo>/<item>`. Pin dependency refs individually when builds
  must be reproducible.
- Validate source registries with `shadcn registry validate` before publishing.

## Programmatic API guardrails

- Import only documented subpaths such as `shadcn/schema`, `shadcn/registry`,
  and `shadcn/preset`. CLI command internals are not stable APIs.
- Replace `fetchRegistry` with `getRegistry` and `resolveRegistryTree` with
  `resolveRegistryItems`.
- Registry reads cache resolved URLs for the process lifetime and coalesce
  concurrent requests. Set `useCache: false` for servers or watchers that need
  fresh data.
- `addRegistryItems` throws instead of exiting and does not load project config.
  Resolve and pass configuration containing aliases and `resolvedPaths` unless
  every payload file has an explicit universal target.
- Catch typed `RegistryError` failures or their specific subclasses. Do not parse
  process output to distinguish missing items, authentication, fetch,
  configuration, file, parsing, validation, namespace, or environment errors.

## Choose the right reference

- For a framework initialization, Tailwind upgrade, theme refresh, or preset,
  read [Project setup and styling](references/project-setup-and-styling.md).
- For primitive-base behavior, Toast, chat demos, rendered content, or staged
  Radix migration, read [Components and bases](references/components-and-bases.md).
- For command selection, dry runs, MCP, docs, or source rewrites, read
  [CLI and migrations](references/cli-and-migrations.md).
- For registry authoring, private access, GitHub distribution, schemas, direct
  API consumption, or preset tooling, read
  [Registries and programmatic APIs](references/registries-and-programmatic-apis.md).
