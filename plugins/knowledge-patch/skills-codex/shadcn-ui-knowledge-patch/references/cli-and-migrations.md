# CLI Workflows and Migrations

## Install the shadcn skill

The shadcn skill supplies current Radix and Base UI primitive APIs, component
patterns, registry workflows, CLI usage, and project-aware design-system
guidance.

```sh
pnpm dlx skills add shadcn/ui
```

This facility comes from `cli-v4-and-presets`.

## Inspect the project and component documentation

`info` reports the detected framework and version, CSS-variable setup,
installed components, and component resources. Prefer JSON for automation:

```sh
pnpm dlx shadcn@latest info --json
```

`docs` returns documentation, examples, and primitive API references for a
component. Supply a base when the answer must not depend on project detection:

```sh
pnpm dlx shadcn@latest docs combobox --base radix --json
```

Both commands and their JSON output come from `cli-v4-and-presets`.

## Preview component changes before writing

The `add` command has three preflight flags:

- `--dry-run` prints the planned registry changes without writing files.
- `--diff` compares an installed primitive against its registry update.
- `--view` exposes the registry item before installation.

```sh
pnpm dlx shadcn@latest add button --dry-run
pnpm dlx shadcn@latest add button --diff
pnpm dlx shadcn@latest add button --view
```

Use `--diff` before overwriting locally customized components. These flags are
from `cli-v4-and-presets`.

## Discover decentralized registry items

CLI 3.0 added commands to inspect and search registries:

```sh
pnpm dlx shadcn view @acme/auth-system
pnpm dlx shadcn search @tweakcn -q "dark"
pnpm dlx shadcn list @acme
```

- `view` inspects one item.
- `search` queries the registry's items.
- `list` enumerates a registry.

These discovery commands come from `cli-3-and-mcp`.

## Initialize the project MCP server

One command initializes the project's MCP server:

```sh
pnpm dlx shadcn@latest mcp init
```

The server works with every registry configured in the project, including
several registries at once. This behavior comes from `cli-3-and-mcp`.

## Migrate icon libraries

`migrate icons` rewrites imports and JSX, installs the target icon library, and
updates `components.json`:

```sh
pnpm dlx shadcn@latest migrate icons --from lucide --to phosphor --yes
```

A path or glob scopes the source rewrite, but a scoped run does not update
`components.json`. Icons without a target match are left in place and reported.

## Enable RTL

`migrate rtl` enables RTL, converts physical CSS utilities to logical forms,
and adds directional variants where necessary. It can be scoped to a path or
glob:

```sh
pnpm dlx shadcn@latest migrate rtl "src/components/ui/**"
```

Review customized layout utilities after the rewrite.

## Consolidate Radix dependencies

`migrate radix` changes primitive imports to the unified `radix-ui` package:

```sh
pnpm dlx shadcn@latest migrate radix
```

After the migration succeeds, remove unused individual Radix packages. Icon,
RTL, and Radix migrations are all from `cli-v4-and-presets`.

## Eject shared Tailwind utilities only deliberately

New projects import `shadcn/tailwind.css` for shared Tailwind v4 variants,
utilities, and animations. The following command irreversibly inlines that CSS
and removes the `shadcn` dependency:

```sh
pnpm dlx shadcn@latest eject
```

Once ejected, later CLI updates to the shared stylesheet do not apply
automatically. Commit first and treat the copied CSS as application-owned.
