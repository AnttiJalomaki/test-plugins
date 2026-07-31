# Async Reactivity and State

Use this reference for computed values, tasks, stores, tracking, and reactive
notification behavior.

## Async computed functions

*Batch: `v1.8-1.13`*

Async callbacks to `useComputed$` are deprecated and will stop working in the
next major generation. Their dependency tracking is incomplete: a signal first
read after an `await` is not tracked. Returning an initial promise also
restarts rendering.

Move asynchronous work to `useTask$` or `useResource$`.

Before migrating, inspect every signal read in the callback. A callback that
appears to rerun correctly can still miss dependencies that are first touched
after asynchronous control resumes.

## `useTask$` eagerness

*Batch: `v1.8-1.13`*

The `eagerness` option of `useTask$` is deprecated as of 1.13 and is removed
in the next major generation. Remove the option instead of building new task
scheduling around it.

## Raw store access

*Batch: `v1.8-1.13`*

`unwrapStore()` exposes the underlying content of a store. Use it when a
platform API needs plain data rather than a reactive proxy, including
structured cloning and IndexedDB storage.

```ts
import { unwrapStore } from '@builder.io/qwik';

const copy = structuredClone(unwrapStore(store));
```

Use this low-level API when the receiving operation needs the store's
underlying content.

## Reactive store membership

*Batch: `v1.8-1.13`*

The `in` operator participates in tracking for stores:

```ts
const hasStatus = 'status' in store;
```

When this expression runs in a reactive consumer, it subscribes to the
presence of that property. Adding or removing the property can rerun the
consumer even if no property value was read.

Review code that used membership checks as supposedly untracked probes. Wrap
the read in the appropriate untracked flow when a subscription is unwanted.

## Expanded `untrack()`

*Batch: `v1.14-1.19`*

`untrack()` accepts a signal or store directly:

```ts
const signalValue = untrack(signal);
const storeValue = untrack(store);
```

Its callback form also accepts arguments:

```ts
const total = untrack((a, b) => a + b, 1, 2);
```

Use these forms to read reactive values without subscribing the current
consumer.

## Computed-signal notifications

*Batch: `v1.14-1.19`*

Computed signals notify listeners only when the computed result changes. A
dependency update that produces an equivalent result no longer triggers those
listeners.

Tests should assert effects from meaningful computed-value changes, not from
every upstream write. If code relied on an upstream write as an implicit
notification pulse, model that pulse explicitly instead of depending on an
unchanged computed value.
