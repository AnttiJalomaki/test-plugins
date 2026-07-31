# Reactivity and Async Computations

## Async computed callbacks

Async callbacks to `useComputed$` are deprecated and stop working in V2. A
signal first read after an `await` is not tracked, and returning an initial
promise restarts rendering. Put asynchronous work in `useTask$` or
`useResource$` instead.

## Task eagerness

The `eagerness` option on `useTask$` is deprecated in 1.13 and is removed in
V2. Remove the option rather than carrying it into migrated code.

## Reactive store membership

The membership expression below creates a subscription:

```ts
const exists = 'prop' in store;
```

A reactive consumer that evaluates it tracks changes to whether `prop` is
present, not just reads of the property's value.

## Reading without subscriptions

`untrack()` accepts a signal or store directly. Its callback form also accepts
arguments:

```ts
const value = untrack(signal);
const rawStore = untrack(store);
const result = untrack((a, b) => a + b, 1, 2);
```

Use these forms when the read must not create a reactive subscription.

## Computed-signal notifications

Computed signals notify listeners only when their computed value changes. If a
dependency changes but produces an equivalent result, listeners are not
notified.
