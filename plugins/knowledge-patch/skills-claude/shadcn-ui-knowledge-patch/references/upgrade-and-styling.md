# Upgrade and Styling

## Keep Stack Upgrades Explicit

The CLI can initialize Tailwind v4 and React 19 projects. Existing Tailwind v3
and React 18 applications stay on their installed stack, including when the CLI
adds more components. Adding a component is not an upgrade operation.

Before moving to Tailwind v4, confirm that its browser support matches the
application's requirements and run the upgrade codemod:

```sh
npx @tailwindcss/upgrade@next
```

Review the codemod result separately from shadcn/ui component changes.

## Tailwind v4 Variables

Move `:root` and `.dark` out of `@layer base`. A palette variable contains the
complete color expression. `@theme inline` maps that variable directly and
must not add another `hsl()`, `oklch()`, or other color-function wrapper.

```css
:root {
  --background: oklch(1 0 0);
  --foreground: oklch(0.145 0 0);
}

.dark {
  --background: oklch(0.145 0 0);
  --foreground: oklch(0.985 0 0);
}

@theme inline {
  --color-background: var(--background);
  --color-foreground: var(--foreground);
}
```

New palettes use OKLCH. The same complete-value rule applies to chart colors:

```ts
const chartConfig = {
  revenue: { label: "Revenue", color: "var(--chart-1)" },
}
```

## React 19 Wrappers and Slots

Current generated components no longer need `React.forwardRef` wrappers. Type
the function with `React.ComponentProps<typeof Primitive>`, pass all remaining
props through, and allow `ref` to travel with those props. Add `data-slot` to
every rendered primitive so styling can address stable component parts. Remove
the old wrapper's `displayName` assignment.

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

Apply this as a generated-source pattern, not as a reason to rewrite unrelated
application components.

## Deprecated Components and Defaults

The older `toast` component is deprecated in favor of `sonner`. Base UI's newer
Toast is a separate implementation and remains available for Base UI projects;
identify the project's base before deciding which advice applies.

The old `default` component style is deprecated, and new projects use
`new-york`. Installed source stays local until explicitly regenerated. Buttons
now preserve the browser's default cursor instead of forcing a pointer.

## Animation Package Replacement

Replace `tailwindcss-animate` with `tw-animate-css` in existing projects:

1. Remove the `tailwindcss-animate` dependency.
2. Remove its `@plugin` directive from CSS.
3. Install `tw-animate-css` as a development dependency.
4. Import the replacement from global CSS.

```sh
pnpm remove tailwindcss-animate
pnpm add -D tw-animate-css
```

```css
@import "tw-animate-css";
```

## Opt In to the Refreshed Dark Palette

The refreshed dark-mode palette applies automatically to projects created on
Tailwind v4, but not to applications upgraded from v3. To opt an upgraded
application in:

1. Commit all local component changes.
2. Overwrite the generated components.
3. Replace the `.dark` variables in global CSS with the refreshed OKLCH values.
4. Review the diff and reapply intentional customizations.

```sh
pnpm dlx shadcn@latest add --all --overwrite
```

Do not run the overwrite against uncommitted customized components.
