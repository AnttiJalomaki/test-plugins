# Components, JSX, events, and testing

## MDX components and layouts

Imported MDX content accepts a `components` prop for replacing element or
component implementations. JavaScript expressions in the MDX can read props,
and a default-exported MDX layout component is honored.

```tsx
import { component$ } from '@builder.io/qwik';
import Content from './markdown.mdx';
import MyComponent from './my-component';

export default component$(() => (
  <Content components={{ MyComponent }} />
));
```

## Error boundaries

Use the `ErrorBoundary` component for rendering failures. The corresponding
`useErrorBoundary` behavior was corrected in Qwik 1.13, so tests written around
earlier behavior should be rechecked after upgrading.

Router status pages are separate from component error boundaries; see
[router-and-navigation.md](router-and-navigation.md) for `404.tsx` and
`error.tsx` layout behavior.

## Promise attributes and spread bindings

JSX attributes may receive promises directly:

```tsx
const src = Promise.resolve('/logo.svg');
return <img src={src} />;
```

Bindings work when passed through spread props, including `bind:checked` and
`bind:value`:

```tsx
const value = useSignal('');
const props = { 'bind:value': value };
return <input {...props} />;
```

## Event-name normalization

JSX event names use these matching rules:

| JSX form | Event name |
| --- | --- |
| `onCustomEvent$` | `customevent` |
| `on-CustomEvent$` | `CustomEvent` |
| `onCustom-Event$` | `custom-event` |

Without a leading `-`, the name is lowercased. A leading `-` preserves the
remaining case. Later dashes remain literal rather than camel-casing the next
letter.

Event-handler types are stricter about scope and no longer claim unsupported
handlers such as `document:OnQVisible$` or `onQIdle$`.

## Event directives

The event segment of `preventdefault:event` and `stoppropagation:event` must be
kebab-case. This makes custom-event directives unambiguous instead of relying
on the mostly lowercase names of DOM events.

JSX handlers also support matching passive and capture markers. For example:

```tsx
<div passive:touchmove onTouchMove$={handler} />
<button capture:click onClick$={handler}>Save</button>
```

Do not combine default prevention with a passive listener unless the behavior
has been deliberately accounted for. Suppress a false-positive optimizer
diagnostic only for the affected line:

```tsx
// @qwik-disable-next-line preventdefault-passive-check
<div passive:touchmove preventdefault:touchmove onTouchMove$={handler} />
```

## Framework lifecycle events

Qwik emits a `CustomEvent` named `qviewTransition` when a view transition
starts. It emits `qrender` after every render.

The qwikloader reruns `qidle` and `qinit` handlers on components rerendered with
those handlers; they are not limited to handlers present during the first page
load.

Containers inserted at runtime are supported. The loader dispatches their
`qinit`, `qidle`, and `qvisible` events when the corresponding lifecycle state
applies.

The V2 loader cannot resume V1 containers. If a document deliberately mixes
generations, load the V1 loader as well.

## Scoped-style selectors

Generated scoped-style class names changed their prefix from `⭐️` to `⚡️`.
Update application CSS or tests that directly select generated names such as
`.⭐️MyComponent` to use `.⚡️MyComponent`.

Prefer selectors that do not depend on generated class names when possible.

## Router test doubles

`QwikCityMockProvider` can mock route loaders and actions. Use those mocks to
exercise component loading, success, explicit `fail()`, and thrown-error paths
without constructing an entire request pipeline.
