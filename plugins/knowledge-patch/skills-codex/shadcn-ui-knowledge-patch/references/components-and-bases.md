# Components and Primitive Bases

## Select and preserve a component base

New projects default to Base UI. Non-interactive scripts and CI pipelines that
expect Radix must request it explicitly:

```sh
pnpm dlx shadcn init -b radix
```

The long form is `--base radix`. A registry author that needs to pin a base
should provide a `registry:base` item. A registry without one initializes as
Base UI under the current default. This behavior comes from
`current-component-bases`.

Base UI retains the shadcn/ui abstraction: application imports, appearance, and
component behavior stay consistent while the underlying primitives change.
When components are added, the CLI detects the project's selected library and
applies the corresponding transformations. That transformation also applies to
components from remote registries, and Base UI remains compatible with existing
components.

```tsx
import { Dialog, DialogContent, DialogTrigger } from "@/components/ui/dialog"
```

The selectable Base UI behavior is captured in `create-and-base-ui`.

## Keep React Aria isolated

The React Aria base supports all eight styles: Vega, Nova, Maia, Lyra, Mira,
Luma, Rhea, and Sera. Its dependencies and state selectors come from the
Aria-specific registry. Installing Aria components does not change existing
Base UI or Radix components.

## Choose the correct toast implementation

The older general `toast` component is deprecated in favor of `sonner`. Base UI
projects now also have a distinct Base UI Toast. That newer component supports:

- actions;
- status types;
- promise state;
- stacked notifications; and
- swipe dismissal.

Install the Base UI implementation's source with the CLI:

```sh
pnpm dlx shadcn@latest add toast
```

The deprecation is from `tailwind-v4-transition`; the base-specific replacement
is from `current-component-bases`. Check the project's base before interpreting
an existing `toast` component.

## Migrate customized Radix components progressively

The shadcn skill can migrate one customized component and all of its usages at a
time while Radix and Base UI coexist, or it can migrate the whole project. A
request can be phrased as:

```text
migrate accordion to base-ui
```

For each component, the workflow:

1. Performs mechanical API changes such as converting `asChild` to `render`.
2. Flags behavior differences that need manual review.
3. Typechecks and builds the project.
4. Writes a component report under `.migration/`.
5. Creates one commit per component on a migration branch.

This component-by-component workflow is from `current-component-bases`. Review
each report and commit before proceeding to the next customized primitive.

## Build deterministic chat demonstrations

`@shadcn/helpers` can drive AI SDK or TanStack AI chat interfaces without an
inference backend, API route, network request, or API key. This is useful for
deterministic demos and component development.

The AI SDK adapter supplies native messages and a `useChat` transport:

```tsx
import { useChat } from "@ai-sdk/react"
import { createChat } from "@shadcn/helpers/ai-sdk"

const chat = createChat()
  .user("What changed?")
  .assistant("Keyboard shortcuts and faster search.")

export function useDemoChat() {
  return useChat({
    messages: chat.get(0),
    transport: chat.transport(),
  })
}
```

The `@shadcn/helpers/tanstack-ai` entry point supplies a TanStack AI `useChat`
connection with real AG-UI events. Scripted conversations can emit delays,
reasoning, tool calls and outputs, sources, and streaming text. These helper
capabilities come from `current-component-bases`.

## Style HTML and rendered Markdown with Typeset

`shadcn/typeset` is a one-file CSS system. Add the `typeset` class to HTML or
rendered Markdown to inherit the active theme and container size. It is safe for
streaming content. Tune each context independently with CSS variables for size,
leading, and flow:

```css
.typeset-chat {
  --typeset-leading: 1.6;
  --typeset-flow: 1em;
}

.typeset-docs {
  --typeset-size: 15px;
  --typeset-leading: 1.75;
  --typeset-flow: 1.5em;
}
```

```tsx
<div className="typeset typeset-chat">{message}</div>
<article className="typeset typeset-docs">{page}</article>
```

Typeset is part of `current-component-bases`.
