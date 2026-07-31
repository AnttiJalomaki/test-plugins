# JSX, Events, and Serialization

## Raw store access

Use `unwrapStore()` when an API needs the store's underlying content, including
structured cloning and IndexedDB operations:

```ts
import { unwrapStore } from '@builder.io/qwik';

const copy = structuredClone(unwrapStore(store));
```

Keep normal component reads on the reactive store; unwrap only for the boundary
that requires raw content.

## MDX component mapping and layouts

Imported Qwik City MDX content accepts a `components` prop for custom component
bindings. JavaScript expressions inside MDX can use props, and Qwik honors an
MDX layout component exported as the default.

```tsx
import { component$ } from '@builder.io/qwik';
import Content from './markdown.mdx';
import MyComponent from './my-component';

export default component$(() => <Content components={{ MyComponent }} />);
```

## View-transition events

Qwik emits a `CustomEvent` named `qviewTransition` when a view transition
starts. Listen for that event when code must coordinate with the transition.

## Error boundaries

Use the `ErrorBoundary` component to contain component errors. Qwik also
corrected `useErrorBoundary` behavior in 1.13, so code that worked around older
hook behavior should be rechecked before retaining the workaround.
