# Events and controls

## Listener lifecycle

`Evented.on()` returns a `Subscription` rather than the evented object as of
5.0.0. Fluent registration such as `map.on('x', x).on('y', y)` does not work.
Register each listener separately and retain subscriptions that must later be
removed.

```js
const moveSubscription = map.on('move', onMove);
map.on('zoom', onZoom);
moveSubscription.unsubscribe();
```

## Event maps and TypeScript

`MapEventType` is an interface rather than a type alias as of 5.8.0.
Applications can add custom events with declaration merging:

```ts
declare module 'maplibre-gl' {
  interface MapEventType {
    'app:ready': {type: 'app:ready'; payload: string};
  }
}
```

The 6.0.0 event type overhaul makes `Evented` abstract and generic over an
event map. Sources expose `SourceEventType`, and the map event map includes
roll lifecycle events and a typed `style.load` event.

Replace removed `MapDataEvent` references with
`MapSourceDataEvent | MapStyleDataEvent`. Rename `MapLibreZoomEvent` references
to `MapBoxZoomEvent`.

Events are classes in v6, but application code should discriminate them by
their `type` field rather than by `instanceof` (migration-v5-v6):

```js
map.on('move', (event) => {
  if (event.type === 'move') handleMove(event);
});
```

## Style load completion

Since 5.16.0, `setStyle()` emits `style.load` when style JSON is applied as a
diff as well as when the style is completely reloaded. Code that waits for the
updated style can use one path for both cases.

```js
map.once('style.load', onStyleLoad);
map.setStyle(nextStyle);
```

## Geolocation

`GeolocateControl` emits `outofmaxbounds` only while
`trackUserLocation` is enabled as of 5.8.0. Treat the event as part of active
location tracking; do not expect it from one-off geolocation when tracking is
disabled.

## Box zoom

Box zoom configuration exposes `boxZoom.boxZoomEnd` as of 5.20.0. Use it to
customize completion behavior after a Shift-drag selection.

## Reduced motion

Map construction accepts `MapOptions.reduceMotion` as of 5.12.0:

```js
const map = new Map({
  container: 'map',
  reduceMotion: true
});
```

## Popup placement

`Popup` accepts `padding` as of 5.16.0. Automatic placement keeps the popup
away from map-container edges by that amount.

```js
const popup = new Popup({padding: 16});
```

## Marker state

Since 5.20.0, `Marker` and `MarkerOptions` accept numbers as well as strings
for `opacity` and `opacityWhenCovered`.

```js
new Marker({opacity: 1, opacityWhenCovered: 0.25});
```

A marker covered by 3D terrain or a globe receives the
`maplibregl-marker-covered` CSS class, allowing covered-state styling.

