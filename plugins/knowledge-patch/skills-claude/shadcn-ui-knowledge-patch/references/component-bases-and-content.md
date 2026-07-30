# Component Bases and Content

## Base UI Projects

New projects default to Base UI. Non-interactive scripts and CI that require
Radix must request it explicitly:

```sh
pnpm dlx shadcn init -b radix
```

Base UI retains the shadcn/ui abstraction, application imports, appearance,
and behavior; the underlying primitives change. Application code continues to
import local components:

```tsx
import { Dialog, DialogContent, DialogTrigger } from "@/components/ui/dialog"
```

When adding a component, the CLI detects the project's configured library and
applies the corresponding transformations. This also applies to components
from remote registries. Base UI implementations remain compatible with
existing components, allowing a project to migrate progressively.

Registry authors that must pin the base should provide a `registry:base` item.
A registry without one initializes with the current Base UI default.

## Base UI Toast

Base UI projects have a current Toast component with actions, status types,
promises, stacking, and swipe dismissal. Install its source through the CLI:

```sh
pnpm dlx shadcn@latest add toast
```

This is distinct from the older Radix-oriented `toast` component deprecated in
favor of `sonner`.

## React Aria Styles Stay Isolated

The React Aria base supports all eight styles: Vega, Nova, Maia, Lyra, Mira,
Luma, Rhea, and Sera. Aria-specific dependencies and state selectors come from
the Aria registry. Adding them does not rewrite existing Base UI or Radix
components.

## Migrate Customized Radix Components Progressively

The shadcn skill can migrate one customized component and all of its usages at
a time, or migrate an entire project. Radix and Base UI may coexist while work
is in progress.

For each component, the migration can:

- Apply mechanical API changes such as converting `asChild` to `render`.
- Flag behavioral differences for human review.
- Typecheck and build the project.
- Write a component report under `.migration/`.
- Create one commit per component on a migration branch.

An instruction can be as focused as:

```text
migrate accordion to base-ui
```

Keep application behavior review in the loop; primitive APIs can be translated
mechanically, but interaction differences may require judgment.

## Build Deterministic Chat Demos

`@shadcn/helpers` can drive AI SDK or TanStack AI chat interfaces without an
external model, API route, network request, or API key. Scripted conversations
can include delays, reasoning, tool calls and results, sources, and streaming
text.

The AI SDK adapter returns native messages and a `useChat` transport:

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

For TanStack AI, import from `@shadcn/helpers/tanstack-ai`; it supplies a
TanStack `useChat` connection that emits real AG-UI events.

## Style HTML and Markdown With Typeset

`shadcn/typeset` is a one-file CSS system for HTML and rendered Markdown. Add
the `typeset` class to inherit the current theme and container width. It is safe
for streaming content.

Tune size, leading, and vertical flow per context through custom properties:

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
