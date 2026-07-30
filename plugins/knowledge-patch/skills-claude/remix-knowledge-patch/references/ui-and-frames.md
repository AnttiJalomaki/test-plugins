# UI and frames

Use this reference for the evolution of the Remix 3 component API, current
handle and mixin patterns, frame-backed asynchronous UI, headless behaviors,
DOM preservation, and generated style precedence.

## Early handle demonstration

The Remix Jam demonstration used `@remix-run/dom`. A component function was
bound to a `Remix.Handle` as `this`, initialized mutable state, and returned a
render callback. Calling `this.update()` scheduled another render. Events used
an `on` prop containing event descriptors such as a local `tempo(...)` helper.

```tsx
import { tempo } from "./tempo";
import { createRoot, type Remix } from "@remix-run/dom";

function App(this: Remix.Handle) {
  let bpm = 60;

  return () => (
    <button
      on={tempo((event) => {
        bpm = event.detail;
        this.update();
      })}
    >
      BPM: {bpm}
    </button>
  );
}

createRoot(document.body).render(<App />);
```

At the time of that recap, this UI implementation had not been open sourced.
Treat it as design history rather than the current beta signature. Source
batch: `remix-jam-2025`.

## Beta handle and mixin model

The beta imports the UI API from `remix/ui`. A component receives `Handle` as a
normal function argument, keeps state in local variables, and returns its
render callback. Element behavior composes through the `mix` array.

`on()` event mixins pass an abort signal to asynchronous handlers. After any
`await`, check the signal before updating local state or calling
`handle.update()`:

```tsx
import { type Handle, on } from "remix/ui";

function Copy(handle: Handle<{ url: string }>) {
  let copied = false;

  return () => (
    <button
      mix={[
        on("click", async (_, signal) => {
          await navigator.clipboard.writeText(handle.props.url);
          if (signal.aborted) return;
          copied = true;
          handle.update();
        }),
      ]}
    >
      {copied ? "Copied" : "Copy"}
    </button>
  );
}
```

This signature supersedes the demonstration's `this: Remix.Handle` form.
Source batch: `remix-3-beta-preview`.

## UI import consolidation and removals

The old `remix/component*` runtime and `remix/components/*` component paths
were removed. Use:

- `remix/ui` for the public runtime;
- the JSX and server runtime subpaths under `remix/ui`;
- `remix/ui/accordion`, `remix/ui/menu`, and other component-specific paths;
  and
- the component `/primitives` variants for lower-level building blocks.

Beta 5 also removed these component subpaths:

- `glyph`;
- `scroll-lock`;
- `separator`; and
- `theme`.

It removed root helper re-exports including `flashAttribute`,
`hiddenTypeahead`, `onKeyDown`, and `waitForCssTransition`. Do not replace the
old imports by guessing another root export; select a surviving component or
primitive API for the behavior.

## Anchors and context menus

`anchor()` accepts either an element or an `AnchorPoint` coordinate target.
Coordinate anchors are useful when there is no persistent trigger element at
the desired position.

`menu.contextTrigger()` opens a menu at the pointer position from a right-click
while retaining keyboard and submenu behavior. Prefer it over hand-built
pointer-only context-menu handling when those accessibility and nesting
semantics are required.

## Selective hydration and frames

`hydrated()` selectively enables client-side JavaScript for a component.
`Frame` is an iframe-inspired asynchronous UI primitive. Hydrated components
can contain frames, and frames can contain hydrated components.

After a mutation, frame revalidation returns raw HTML. The hybrid reconciler
morphs that HTML into the existing DOM rather than requiring a JSON component
payload.

Current public navigation and frame APIs from `remix/ui` include:

- `navigate`;
- `link`;
- `run`;
- `handle.frames.top`; and
- `handle.frames.get(name)`.

For server rendering, import frame source helpers from `remix/ui/server`:

- `frameSrc`; and
- `topFrameSrc`.

Use the handle's frame accessors when an update belongs to a specific named or
top-level frame rather than assuming all navigation affects the whole page.

## Headless popovers

`remix/ui/popover` supplies context and behavior mixins for markup owned by the
application. Attach `anchor()` and `focusOnHide()` to the trigger. Attach
`surface()` to the panel and control it with `open` and `onHide`.

```tsx
import * as popover from "remix/ui/popover";

function ViewOptions() {
  let open = false;

  return () => (
    <popover.Context>
      <button
        mix={[
          popover.anchor({ placement: "bottom-end" }),
          popover.focusOnHide(),
        ]}
        onClick={() => {
          open = true;
        }}
        type="button"
      >
        View options
      </button>
      <div
        mix={[
          popover.surface({
            open,
            onHide() {
              open = false;
            },
          }),
        ]}
      >
        Panel content
      </div>
    </popover.Context>
  );
}
```

Because the application owns the elements, keep their native semantics—such as
button type and accessible naming—alongside the behavior mixins.

## Preserve client-owned DOM during frame updates

Add `rmx-preserve-dom` to the smallest custom element or widget container whose
live attributes and children must survive later frame reconciliation:

```tsx
<pagefind-ui data-key="search" rmx-preserve-dom>
  <button type="button">Search</button>
</pagefind-ui>
```

Its children are still rendered during server rendering, and initial client
entries still hydrate. The attribute protects later client-owned DOM from
replacement; it does not suppress initial rendering or hydration.

Avoid placing it on an unnecessarily large ancestor because everything inside
that boundary becomes client-owned during later frame updates.

## Generated CSS cascade layer

Rules produced by `css(...)` are emitted in the `rmx` cascade layer. Cascade
order consequences are:

- unlayered author rules outrank rules in `rmx`; and
- named layers declared before `rmx` have lower priority than `rmx`.

Declare the intended named-layer order explicitly when integrating framework
styles:

```css
@layer base, rmx;
```

If a generated rule unexpectedly wins or loses, inspect whether the competing
rule is unlayered and where its named layer appears in the layer order before
increasing selector specificity.
