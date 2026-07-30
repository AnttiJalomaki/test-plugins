# UI and frames

## Recognize the component API transition

The initial UI demonstration in `remix-jam-2025` bound the component function
to `this: Remix.Handle`, used an `on` prop with event descriptors, and called
`this.update()`:

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

At the time, that UI implementation was not yet open source. This form is
useful for identifying old examples, but the beta API uses a normal handle
argument and `mix`.

## Write beta components with handles and mixins

In `remix-3-beta-preview`, a component receives its `Handle` as an ordinary
argument, initializes mutable local state, and returns a render callback.
Element behavior composes through the `mix` array:

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

The abort check is part of the lifecycle contract. Do it after awaited work
before mutating state or calling `handle.update()`.

## Migrate UI imports

The old `remix/component*` runtime and `remix/components/*` paths are removed.
Use:

- `remix/ui` for the component runtime.
- The JSX and server runtime subpaths under `remix/ui`.
- Component paths such as `remix/ui/accordion` and `remix/ui/menu`.
- A component's `/primitives` variant when working with its lower-level
  building blocks.

By beta 5, `glyph`, `scroll-lock`, `separator`, and `theme` subpaths were also
removed. Root helper re-exports including `flashAttribute`,
`hiddenTypeahead`, `onKeyDown`, and `waitForCssTransition` are absent.

`anchor()` accepts either an element or an `AnchorPoint` coordinate target.
`menu.contextTrigger()` opens at a right-click pointer location while
preserving keyboard and submenu behavior.

## Hydrate selectively

`hydrated()` enables client-side JavaScript for an individual component rather
than requiring the entire document to hydrate. Hydrated components and
`Frame` instances can contain one another.

After a mutation, frame revalidation returns raw HTML. The hybrid reconciler
morphs that response into the existing DOM rather than replacing the entire
application tree.

## Navigate and render frames

The client UI entrypoint exposes:

- `navigate`
- `link`
- `run`
- `handle.frames.top`
- `handle.frames.get(name)`

For SSR frame sources, import `frameSrc` and `topFrameSrc` from
`remix/ui/server`. Keep server source generation separate from the client
navigation helpers.

## Compose a headless popover

`remix/ui/popover` provides context and behavior mixins for markup owned by
the application. Attach `anchor()` and `focusOnHide()` to the trigger, and
control the surface with `open` and `onHide`:

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

The application owns the elements and state; the mixins supply positioning,
dismissal, and focus behavior.

## Preserve client-owned DOM

Add `rmx-preserve-dom` to the smallest custom element or widget container
whose live attributes and children must survive later frame updates:

```tsx
<pagefind-ui data-key="search" rmx-preserve-dom>
  <button type="button">Search</button>
</pagefind-ui>
```

The children still render during SSR, and initial client entries still
hydrate. The marker affects later morphs; it is not a directive to omit the
subtree from initial rendering.

## Order generated CSS intentionally

Rules produced by `css(...)` are emitted in the `rmx` cascade layer.

- Unlayered rules outrank `rmx` rules.
- Named layers declared before `rmx` lose to it.

Declare layer order explicitly when combining generated and application CSS:

```css
@layer base, rmx;
```
