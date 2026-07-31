# Components and Events

Use this reference for MDX composition, error handling, view-transition
events, and Qwik City test doubles.

## MDX component injection and layouts

*Batch: `v1.8-1.13`*

Imported Qwik City MDX content accepts a `components` prop. Map names used by
the document to application components:

```tsx
import { component$ } from '@builder.io/qwik';
import Content from './markdown.mdx';
import MyComponent from './my-component';

export default component$(() => (
  <Content components={{ MyComponent }} />
));
```

JavaScript expressions in MDX can use props. A default-exported MDX layout
component is also honored, so do not wrap the document a second time unless
the application intentionally needs another layout layer.

## View-transition event

*Batch: `v1.8-1.13`*

Qwik emits a `CustomEvent` named `qviewTransition` when a view transition
starts. Use that event for application behavior that must synchronize with the
start of a transition.

Keep the exact lowercase-leading event name. This is a Qwik lifecycle event,
not the browser's generic view-transition API.

## Error boundaries

*Batch: `v1.8-1.13`*

Use the `ErrorBoundary` component to contain rendering failures. The behavior
of `useErrorBoundary` was corrected in 1.13, so update tests that encoded an
older workaround before retaining that workaround.

Server-function and route-loader failures have a separate transport flow; see
[Server and route data](server-and-route-data.md#server-function-error-flow).

## Route-loader and action mocks

*Batch: `v1.14-1.19`*

`QwikCityMockProvider` can mock route loaders and actions in component and
integration tests. Supply test values through the provider rather than
requiring a live Qwik City request pipeline.

Exercise both successful data and failure/action-result paths. Provider-based
mocks should reflect the signal and result shapes consumed by the component.
