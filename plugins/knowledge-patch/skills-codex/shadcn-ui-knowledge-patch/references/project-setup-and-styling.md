# Project Setup and Styling

## Preserve the current stack during upgrades

The CLI can initialize projects with Tailwind v4 and React 19, but an existing
Tailwind v3 or React 18 application stays on that stack until it is explicitly
upgraded. Adding components does not cross that boundary.

Before upgrading Tailwind, verify that the application's browser support is
compatible with Tailwind v4 and run the preview upgrade codemod:

```sh
npx @tailwindcss/upgrade@next
```

This transition behavior comes from the `tailwind-v4-transition` batch.

## Lay out Tailwind v4 variables

Move `:root` and `.dark` outside `@layer base`. Each custom property should hold
the full color expression. Map it to Tailwind through `@theme inline` without
adding a second color-function wrapper.

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

New palettes use OKLCH. Code that consumes chart tokens should also use the full
variable directly:

```ts
const chartConfig = {
  series: {
    color: "var(--chart-1)",
  },
}
```

Do not rewrite that value as a variable fragment nested in `hsl()` or `oklch()`.

## Opt in to the refreshed dark palette

The March 2025 dark-mode palette is included in projects created on Tailwind v4.
It is not automatically applied to projects upgraded from v3. To adopt it:

1. Commit local component changes so the overwrite has a clean comparison point.
2. Overwrite the project components.
3. Replace the dark variables in `globals.css` with the new OKLCH values.
4. Review the diff and reapply intentional customizations.

```sh
pnpm dlx shadcn@latest add --all --overwrite
```

Treat this as a palette migration, not an incidental consequence of installing
Tailwind v4.

## Replace the animation package

`tailwindcss-animate` is deprecated for new projects. In an existing project:

1. Remove the `tailwindcss-animate` dependency.
2. Remove its `@plugin` directive.
3. Install `tw-animate-css` as a development dependency.
4. Import it from the global stylesheet.

```css
@import "tw-animate-css";
```

## Scaffold a project

The current CLI's `init` command can scaffold Next.js, Vite, TanStack Start,
React Router, Astro, or Laravel. `create` is an alias for `init`. Use `--name`
for a new project directory, `--monorepo` for a workspace, and `--base` to select
Base UI, Radix, or Aria primitives.

```sh
pnpm dlx shadcn@latest init --name dashboard --template astro --base radix
pnpm dlx shadcn@latest init --template next --monorepo
```

The earlier interactive creation workflow also supports a v0 target and prompts
for the component library, icon set, base color, theme, fonts, and visual style:

```sh
npx shadcn create
```

The five supplied style choices in that workflow are:

| Style | Character |
| --- | --- |
| Vega | Classic |
| Nova | Reduced spacing |
| Maia | Soft and rounded |
| Lyra | Boxy and sharp |
| Mira | Compact and dense |

A style selection rewrites component code, including fonts, spacing, structure,
and supporting libraries. It is not merely a color-theme switch. These creation
details come from `create-and-base-ui`; current initialization behavior comes
from `cli-v4-and-presets`.

## Apply portable presets

A preset code packages colors, theme, icon library, fonts, and radius. Use it
with `init --preset` to scaffold with that configuration or to switch an
existing application; a full switch also reconfigures its components.

```sh
pnpm dlx shadcn@latest init --preset a1Dg5eFl
```

The `apply` command applies a code to an existing project. Restrict it to theme
or font when UI components must not be reinstalled:

```sh
pnpm dlx shadcn@latest apply a2r6bw --only theme
pnpm dlx shadcn@latest apply a2r6bw --only font
```

Inspect a code with `preset decode`, or reconstruct the project's current code
with `preset resolve`. Both support JSON. `preset info` is an alias for
`preset resolve`.

```sh
pnpm dlx shadcn@latest preset decode a2r6bw --json
pnpm dlx shadcn@latest preset resolve --json
pnpm dlx shadcn@latest preset info --json
```

Preset commands and portable codes come from `cli-v4-and-presets`.

## Understand shared Tailwind CSS and ejection

New initialization imports `shadcn/tailwind.css`, which supplies shared Tailwind
v4 variants, utilities, and animations. The `eject` command irreversibly copies
that CSS into the project and removes the `shadcn` dependency:

```sh
pnpm dlx shadcn@latest eject
```

After ejection, the project owns the copied CSS. Later CLI changes to the shared
stylesheet are no longer received automatically. Review and commit the project
before running this command.

## Use current component-source conventions

React 19 component wrappers use functions typed with `React.ComponentProps`
instead of `React.forwardRef`. Pass the ref with the remaining props, add a
`data-slot` marker to every primitive, and remove the old wrapper's `displayName`
assignment.

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

Other changed defaults from `tailwind-v4-transition` are:

- The legacy `toast` is deprecated in favor of `sonner`.
- The `default` style is deprecated; new projects use `new-york`.
- Buttons retain the browser's default cursor.
