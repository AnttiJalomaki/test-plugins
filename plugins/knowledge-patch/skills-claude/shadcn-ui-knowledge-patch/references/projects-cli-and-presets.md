# Projects, CLI, and Presets

## Create and Initialize Projects

`init` can scaffold Next.js, Vite, TanStack Start, React Router, Astro, and
Laravel projects. `create` is an alias. Use `--name` for a new project,
`--monorepo` for a workspace, and `--base` to choose Base UI, Radix, or Aria
primitives.

```sh
pnpm dlx shadcn@latest init --name dashboard --template astro --base radix
pnpm dlx shadcn@latest init --template next --monorepo
```

The interactive creator configures a Next.js, Vite, TanStack Start, or v0
project and asks for the component library, icons, base color, theme, fonts,
and visual style:

```sh
npx shadcn create
```

The supplied styles are:

- Vega: classic.
- Nova: reduced spacing.
- Maia: soft and rounded.
- Lyra: boxy and sharp.
- Mira: compact and dense.

A style selection rewrites component code, including fonts, spacing, structure,
and supporting libraries. It is broader than a color-theme switch.

## Use Portable Preset Codes

A preset code packages colors, theme, icon library, fonts, and radius.
`init --preset` can scaffold with the code or switch an existing application,
reconfiguring its components as well.

```sh
pnpm dlx shadcn@latest init --preset a1Dg5eFl
```

`apply` changes an existing project. Restrict it with `--only theme` or
`--only font` when UI components should not be reinstalled.

```sh
pnpm dlx shadcn@latest apply a2r6bw --only theme
```

Inspect codes and reconstruct the current project's code with the preset
commands. Both support JSON output, and `preset info` aliases `preset resolve`.

```sh
pnpm dlx shadcn@latest preset decode a2r6bw --json
pnpm dlx shadcn@latest preset resolve --json
pnpm dlx shadcn@latest preset info --json
```

## Preflight Component Changes

The `add` command can expose its changes before writing:

```sh
pnpm dlx shadcn@latest add button --dry-run
pnpm dlx shadcn@latest add button --diff
pnpm dlx shadcn@latest add button --view
```

Use `--dry-run` for the planned operation, `--view` for registry content, and
`--diff` to compare an installed primitive with its registry update. A diff is
especially important before merging an update into locally customized source.

## Inspect Project-Aware Context

`info` reports the framework and version, CSS-variable setup, installed
components, and component resources. `docs` returns documentation, examples,
and primitive API references for a component; pass a base when it must differ
from project context. Both commands can emit JSON.

```sh
pnpm dlx shadcn@latest info --json
pnpm dlx shadcn@latest docs combobox --base radix --json
```

## Run Built-in Migrations

### Icons

`migrate icons` rewrites imports and JSX, installs the destination icon library,
and updates `components.json`:

```sh
pnpm dlx shadcn@latest migrate icons --from lucide --to phosphor --yes
```

A path or glob scopes the source rewrite but does not update `components.json`.
Icons without a destination match are left unchanged and reported.

### Right-to-left layout

`migrate rtl` enables RTL, changes physical CSS utilities to logical utilities,
and adds directional variants where needed:

```sh
pnpm dlx shadcn@latest migrate rtl "src/components/ui/**"
```

### Radix package consolidation

`migrate radix` switches imports to the unified `radix-ui` package. After the
migration and verification, remove individual primitive packages that are no
longer imported.

```sh
pnpm dlx shadcn@latest migrate radix
```

## Shared Tailwind Utilities and Ejection

New initialization imports `shadcn/tailwind.css`, which provides shared
Tailwind v4 variants, utilities, and animations. `eject` copies that CSS into
the application and removes the `shadcn` dependency:

```sh
pnpm dlx shadcn@latest eject
```

Ejection is irreversible from the CLI's perspective. Future updates to the
shared stylesheet will not flow into the copied CSS automatically.

## Coding Skill and Project MCP

Install the shadcn skill to provide automated coding workflows with Radix and
Base UI APIs, component patterns, registry procedures, and project-aware CLI
usage:

```sh
pnpm dlx skills add shadcn/ui
```

Initialize the project MCP server with one command. It operates across every
registry configured for the project, including several registries together.

```sh
pnpm dlx shadcn@latest mcp init
```
