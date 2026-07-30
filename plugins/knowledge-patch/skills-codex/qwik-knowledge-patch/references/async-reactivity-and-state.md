# Async computation, reactivity, and state

## Choosing an async primitive

The recommended primitive changed during the V2 prereleases. Keep the target
generation clear:

- In V1, async callbacks to `useComputed$()` are deprecated and will not work
  in V2-era semantics. Signals first read after `await` are not tracked, and an
  initial promise restarts rendering. Use `useTask$()` or `useResource$()` in a
  V1 application.
- Early V2 introduced `useAsync$()` as the async-computation replacement and
  added explicit invalidation plus configurable `always` or `never`
  serialization.
- Current V2 accepts async functions in `useComputed$()`. `useAsync$()`,
  `createAsync$()`, and `AsyncSignal` are deprecated in its favor.

Current computed signals accept a compute context with `track()`. Use it for a
dependency first read after an `await`. The `clientOnly` option skips server
computation. Failures are exposed through `.error`, and reading `.value` for a
failed computation rethrows.

```ts
const data = useComputed$(async ({ abortSignal, track }) => {
  const id = track(idSignal);
  const response = await fetch(`/api/items/${id}`, { signal: abortSignal });
  return response.json();
});
```

## Migrating resources

`useResource$()` and `<Resource />` are deprecated in favor of the async
computed model. During migration:

- Automatic signal tracking covers reads before the first `await`; use context
  `track()` for reads after it.
- Use the context `abortSignal` instead of manual cancellation cleanup.
- Read `.value` as `T`, not `Promise<T>`.
- Test `.pending` before reading unresolved `.value`, because that read throws.
- Read failures through `.error`.
- Use the context's `previous` value when an update depends on the prior result.
- Configure `initial`, `expires`, and `poll` for startup and refresh behavior.
- Set `concurrency: 0` when resource-style unlimited parallel work is required.

```tsx
export const Item = component$(() => {
  const idSignal = useSignal('42');
  const data = useComputed$(async ({ abortSignal }) => {
    const response = await fetch(`/api/items/${idSignal.value}`, {
      signal: abortSignal,
    });
    return response.json() as Promise<{ name: string }>;
  }, { concurrency: 0 });

  if (data.pending) return <p>Loading...</p>;
  if (data.error) return <p>{data.error.message}</p>;
  return <p>{data.value.name}</p>;
});
```

## Compatibility behavior for `useAsync$`

When maintaining code that has not yet migrated, use these final semantics:

- `.promise()` replaced `.resolve()`, and later returns `Promise<void>` rather
  than the calculated value. Consume the result from `.value` and failures from
  `.error`.
- An async-signal error is thrown only once.
- Writable results, an optional initial value, eager cleanup, and `clientOnly`
  loading at document-idle are supported.
- The former mutable `interval` polling option was renamed to `expires`; use
  `poll` to request automatic reruns after expiry.
- An interval created during SSR resumes polling on the client.
- `concurrency` defaults to `1` and queues excess work; `0` is unlimited.
- A stale completion does not overwrite a newer result that already completed.
- Starting a new computation triggers the previous computation's
  `abortSignal`; calling `.abort()` cancels the current one and runs cleanup.
- `invalidate(info)` forwards the invalidation reason to the calculation.
- The default serialization strategy became `always`.

Early V2 async signals also expose `loading` and `error`. Computed-like signals
can be invalidated explicitly even when their serialized-value policy is
`never`.

## Signal control

Signals expose `.untrackedValue` for reads and writes that must not create a
subscription. After an untracked write or in-place mutation, call `.trigger()`
to run subscribers explicitly:

```ts
const current = signal.untrackedValue;
signal.untrackedValue = next;
signal.trigger();
```

The optimizer permits an `AsyncSignal` computation to refer to and write to
itself.

Computed signals notify listeners only when the computed result changes. A
dependency update that produces an equal result does not notify them.

V2 var props participate in additional reactive updates, so changes propagate
more consistently to consuming components.

## Stores and untracked access

The expression `"prop" in store` creates a subscription and reacts when the
property's presence changes.

Use `unwrapStore()` when an API needs the underlying object, such as structured
cloning or IndexedDB:

```ts
import { unwrapStore } from '@builder.io/qwik';

const copy = structuredClone(unwrapStore(store));
```

`untrack()` accepts a signal or store directly, or a callback plus arguments:

```ts
const value = untrack(signal);
const result = untrack((a, b) => a + b, 1, 2);
```

These reads do not establish reactive subscriptions.

## Custom and platform serialization

`useSerializer$()` and `createSerializer$()` create signals for values that need
custom serialization. A value carrying `NoSerializeSymbol` is omitted. A
`SerializerSymbol` function returns the object literal to serialize.
`SerializationWeakRef` represents a value that may remain unserialized.

Built-in serialization also supports Temporal values.

## Task scheduling and cleanup

`useTask$()` accepts `deferUpdates`:

```ts
useTask$(() => {}, { deferUpdates: true });
```

Its `eagerness` option was deprecated in 1.13 and is removed from V2 usage. The
`eagerness: 'load' | 'idle'` option on `useVisibleTask$()` is also removed in
V2.

Both `useTask$()` and `useVisibleTask$()` await a returned cleanup promise
before the next invocation. If overlapping cleanup and rerun work is intended,
do not return that promise. `render()` does not await `useVisibleTask$`
callbacks; they execute independently as post-flush effects.

After a resumable application resumes, visible tasks wait until their component
is actually visible. In a client-side-rendered application, they still run
immediately.

## Conditional and suspense rendering

The experimental `Show` component chooses between a `then$` branch and an
optional `else$` branch according to `when$`.

Enable experimental Suspense explicitly:

```ts
qwikVite({ experimental: ['suspense'] });
```

On the client, `<Suspense>` displays its fallback after the configured delay.
`showStale` keeps the last resolved content visible while an update is pending.
`Reveal` coordinates sibling boundaries with `parallel`, `sequential`,
`reverse`, or `together` order; its `collapsed` setting hides boundaries that
are waiting. Out-of-order Suspense streaming is also available experimentally
for SSR.
