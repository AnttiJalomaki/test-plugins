# Reactivity and Async Computations

Use this reference for signals, stores, tasks, asynchronous calculations,
resource migration, error boundaries, and experimental rendering coordination.

Relevant source batches are `v1.8-1.13`, `v1.14-1.19`,
`v2-alpha-beta-1-9`, `v2-beta-10-19`, `v2-beta-20-29`,
`v2-beta-30-38`, and `migration-v1-v2`.

## Track the async API progression

In V1, asynchronous callbacks to `useComputed$()` were deprecated because
signal reads after the first `await` were not tracked and an initial promise
restarted rendering. `useTask$()` or `useResource$()` was the safe V1
alternative.

Early V2 introduced `useAsync$()` as the async-computation replacement. It
eventually exposed loading and error state, explicit invalidation, and
always/never value-serialization strategies.

Later V2 moved asynchronous calculations back to `useComputed$()`.
`useAsync$()`, `createAsync$()`, and `AsyncSignal` are now deprecated. Prefer
the current `useComputed$()` behavior rather than stopping at the intermediate
migration.

## Current async `useComputed$()`

An async computation receives a context that includes `track()`,
`abortSignal`, and `previous`:

```tsx
const data = useComputed$(async ({ abortSignal, track }) => {
  const id = track(() => idSignal.value);
  const response = await fetch(`/api/items/${id}`, {
    signal: abortSignal,
  });
  return response.json() as Promise<{ name: string }>;
}, { concurrency: 0 });

if (data.pending) return <p>Loading...</p>;
if (data.error) return <p>{data.error.message}</p>;
return <p>{data.value.name}</p>;
```

Signal reads are automatically tracked only before the first `await`; call
context `track()` for dependencies read later.

The result rules are:

- `.value` has type `T`, not `Promise<T>`;
- unresolved `.value` throws;
- a failed computation is available through `.error`, and reading `.value`
  rethrows it;
- `.pending` distinguishes the unresolved state; and
- `clientOnly` skips server computation.

Options include `initial`, `expires` with `poll`, and `concurrency: 0` for
unlimited parallel work.

## Maintaining intermediate `useAsync$()` code

Intermediate V2 `useAsync$()` supports writable results, an optional initial
value, eager cleanup, a mutable polling interval, and `clientOnly` loading at
document-idle. A polling interval established during SSR resumes on the
client.

Its concurrency behavior is precise:

- `concurrency` defaults to `1`;
- work queues at the limit;
- `0` means unlimited concurrency;
- stale completions do not overwrite a newer completed result;
- the compute function receives an `abortSignal`;
- starting a new computation aborts the previous one; and
- `.abort()` cancels the current computation and runs cleanup.

The result method is `.promise()`, not the earlier `.resolve()`. It returns
`Promise<void>`, not the computed value. Consume `.value` and `.error`; an
async-signal error is thrown only once.

The default serialization strategy became `always`. The option formerly named
`interval` is now `expires`; `poll` controls reruns after expiry, and
`invalidate(info)` passes a reason to the calculation.

## Resource migration

`useResource$()` and `<Resource />` are deprecated. Migrate to async
`useComputed$()`.

Use `abortSignal` rather than manual cleanup. Use `track()` for signal reads
after the first `await`, consume resolved `.value` as `T`, and use `.pending`
and `.error` for state. Use `concurrency: 0` when the old resource behavior
requires parallel executions.

## Direct signal control

Signals expose `.untrackedValue` for reads and writes without subscriptions,
plus `.trigger()` for explicitly running subscribers after an untracked write
or an in-place mutation:

```ts
const current = signal.untrackedValue;
signal.untrackedValue = next;
signal.trigger();
```

The optimizer accepts self-references, so an `AsyncSignal` in intermediate V2
code can write to itself.

## Store tracking and raw values

The expression `"prop" in store` creates a subscription. Reactive consumers
therefore update when the property's presence changes.

Use low-level `unwrapStore()` when an underlying store value must go through
structured cloning or IndexedDB:

```ts
import { unwrapStore } from '@builder.io/qwik';

const copy = structuredClone(unwrapStore(store));
```

Do not use raw content when proxy-aware reactive behavior is required.

## Untracked reads

`untrack()` accepts a signal or store directly. Its callback form accepts
arguments:

```ts
const value = untrack(signal);
const result = untrack((a, b) => a + b, 1, 2);
```

These reads do not establish subscriptions.

## Computed notification semantics

A computed signal notifies listeners only when its computed value changes. A
dependency update that produces an equivalent result does not notify those
listeners.

V2 var props participate in more reactive updates, so changes propagate more
consistently to consuming components.

## Task options and timing

The `eagerness` option of `useTask$()` is deprecated from 1.13 and removed in
V2. `useTask$()` in intermediate V2 also accepts `deferUpdates`.

The `eagerness: 'load' | 'idle'` option of `useVisibleTask$()` is removed in V2
and must be deleted.

After a resumable application resumes, visible tasks run only once their
component is visible. They still run immediately in a client-side-rendered
application.

`useTask$()` and `useVisibleTask$()` await a returned cleanup promise before
the next invocation. Do not return that promise when overlapping executions
are intended.

Conversely, `render()` does not wait for `useVisibleTask$()` callbacks. They
run independently as post-flush effects.

## Error boundaries

Qwik provides an `ErrorBoundary` component. The behavior of
`useErrorBoundary()` was corrected in 1.13. Prefer a boundary when a rendering
subtree needs local failure handling; route and server failures have separate
Router control flow.

## Experimental `Show`

The experimental `Show` component chooses a `then$` branch or optional `else$`
branch from `when$`.

## Experimental Suspense

Enable Suspense explicitly:

```ts
qwikVite({ experimental: ['suspense'] });
```

On the client, `<Suspense>` shows its fallback after the configured delay.
`showStale` retains the last resolved content during an update.

`Reveal` coordinates sibling boundaries. Its order can be `parallel`,
`sequential`, `reverse`, or `together`; `collapsed` hides boundaries that are
still waiting. Out-of-order Suspense streaming is also experimental for SSR.
