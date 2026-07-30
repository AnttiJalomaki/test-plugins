# Events, types, and errors

## Listener subscriptions

Starting in 5.0.0, `Evented.on()` returns a `Subscription` rather than the evented object. Chaining `map.on(...).on(...)` is invalid. Register listeners separately and retain a returned subscription when it needs to be removed.

```js
const moveSubscription = map.on('move', onMove);
map.on('zoom', onZoom);
moveSubscription.unsubscribe();
```

In 6.0.0, `Evented` is abstract and generic over an event map. A typed event base can be declared as:

```ts
abstract class TypedEvents extends Evented<AppEventType> {}
```

## Event typing and discrimination

`MapEventType` became an interface in 5.8.0, enabling TypeScript declaration merging for application events.

```ts
declare module 'maplibre-gl' {
  interface MapEventType {
    'app:ready': {type: 'app:ready'; payload: string};
  }
}
```

Events are represented by classes after the v6 migration, but application code should discriminate them by their `type` field rather than `instanceof`:

```js
map.on('move', (event) => {
  if (event.type === 'move') handleMove(event);
});
```

The 6.0.0 type overhaul also makes these changes:

- Sources expose `SourceEventType`.
- The map event map includes roll lifecycle events and typed `style.load` events.
- `MapDataEvent` is removed; use `MapSourceDataEvent | MapStyleDataEvent`.
- `MapLibreZoomEvent` is renamed to `MapBoxZoomEvent`.

## Style lifecycle

Since 5.16.0, `setStyle()` emits `style.load` when supplied style JSON is applied as a diff, not only when the style is fully reloaded. Code waiting for the updated style can use the same event in both cases.

```js
map.once('style.load', onStyleLoad);
map.setStyle(nextStyle);
```

## Missing style images

The v6 migration changes missing-image resolution. Calling `Map#addImage` from `styleimagemissing` cannot satisfy the request that triggered that event. Register `Map#setMissingStyleImageResolver` instead.

The resolver can be synchronous or asynchronous. For an asynchronous resolver, call `addImage` before its promise settles:

```js
map.setMissingStyleImageResolver(async (id) => {
  const image = await generateImage(id);
  map.addImage(id, image);
});
```

The `styleimagemissing` event remains useful for observing image IDs that remain unresolved, not for fulfilling the current lookup.

## Request and initialization errors

Since 5.0.0, fetch failures—including CORS, DNS, and malformed-URL failures—reach the map's `error` event as `AJAXError` instances. Inspect the request details on that error rather than expecting an untyped fetch exception.

In 6.0.0, failure to obtain required WebGL support is also delivered through the map's `error` event.
