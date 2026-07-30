# JSX, Events, and Serialization

Use this reference for JSX attribute behavior, custom-event matching,
prevention markers, MDX rendering, serialized state, and Qwikloader lifecycle.

Relevant source batches are `v1.8-1.13`, `v2-alpha-beta-1-9`,
`v2-beta-10-19`, `v2-beta-20-29`, `v2-beta-30-38`, and
`migration-v1-v2`.

## JSX event-name matching

Event names without a leading dash are lowercased. A leading dash preserves
the remaining case, and later dashes are preserved rather than causing
camel-case conversion:

| JSX handler | Event name |
| --- | --- |
| `onCustomEvent$` | `customevent` |
| `on-CustomEvent$` | `CustomEvent` |
| `onCustom-Event$` | `custom-event` |

JSX event-handler types are stricter about scope and no longer advertise
unsupported handlers such as `document:OnQVisible$` or `onQIdle$`.

## Event directives and listener markers

The event segment in `preventdefault:event` and `stoppropagation:event` must be
kebab-case. This supports custom event names in addition to mostly lowercase
DOM event names.

JSX handlers recognize passive and capture markers:

```tsx
<div passive:touchmove onTouchMove$={handleMove} />
<button capture:click onClick$={handleClick}>Save</button>
```

The optimizer can suppress a specific next-line diagnostic with
`@qwik-disable-next-line`, including `preventdefault-passive-check`. Keep the
hint narrowly scoped.

## Promise-valued attributes

JSX attributes accept promises directly:

```tsx
const src = Promise.resolve('/logo.svg');
return <img src={src} />;
```

The caller does not need to resolve an asynchronous attribute value before
rendering.

## Bindings through spread props

`bind:checked` and `bind:value` work through spread props:

```tsx
const value = useSignal('');
const props = { 'bind:value': value };
return <input {...props} />;
```

## MDX components and layouts

Imported MDX accepts a `components` prop for custom component substitution.
MDX JavaScript expressions can use props, and default-exported MDX layout
components are honored.

```tsx
import { component$ } from '@builder.io/qwik';
import Content from './markdown.mdx';
import MyComponent from './my-component';

export default component$(() => (
  <Content components={{ MyComponent }} />
));
```

## Scoped style names

V2 generated scoped-style class names use `⚡️` instead of `⭐️`. Update
selectors that directly name generated classes, for example
`.⭐️MyComponent` to `.⚡️MyComponent`. Prefer not to couple application CSS to
generated names.

## Custom serialization

`useSerializer$()` and `createSerializer$()` create signals for
custom-serializable values.

Objects carrying `NoSerializeSymbol` are omitted from serialization. A
`SerializerSymbol` function returns the serializable object literal.
`SerializationWeakRef` represents a value that may remain unserialized.

Qwik's native serializer also supports Temporal values.

## Serialized document state

V2 replaces:

```html
<script type="qwik/json">
```

with separate scripts at the end of the document:

```html
<script type="qwik/vnode"></script>
<script type="qwik/state"></script>
```

Update tooling, CSP logic, scrapers, and tests that locate or parse serialized
state.

## Qwikloader generation compatibility

The V2 qwikloader does not process V1 containers. A page that retains V1
containers must load the V1 qwikloader too.

Rerendered components with `qidle` or `qinit` handlers run those handlers
again; execution is no longer limited to handlers present at initial page
load.

Containers inserted at runtime are supported. Qwikloader runs their `qinit`,
`qidle`, and `qvisible` events as appropriate.

## Render lifecycle events

Qwik emits:

- `qviewTransition`, a `CustomEvent` when a view transition starts; and
- `qrender` after every render.

Visible tasks are separate from the render-completion contract:
`render()` does not wait for `useVisibleTask$()` callbacks, which execute as
post-flush effects.
