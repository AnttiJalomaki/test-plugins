# Controls, markers, and UI

## Geolocation tracking

`GeolocateControl` emits `outofmaxbounds` only while `trackUserLocation` is enabled as of 5.8.0. Treat that event as part of active location tracking; do not expect it from one-shot or disabled tracking.

## Reduced motion

Map construction accepts `MapOptions.reduceMotion` since 5.12.0. Configure it at map level when motion behavior must follow an application accessibility policy.

```js
const map = new Map({
  container: 'map',
  reduceMotion: true
});
```

## Popup placement

`Popup` accepts `padding` as of 5.16.0. It keeps automatic popup placement away from the edges of the map container.

```js
const popup = new Popup({padding: 16});
```

## Box zoom completion

Box-zoom configuration exposes `boxZoom.boxZoomEnd` since 5.20.0. Use it to customize what happens when a Shift-drag box selection completes.

## Marker drag coordinates

Globe marker dragging no longer reports a spurious ±360-degree longitude offset as of 5.4.0. Use the longitude supplied to the drag handler without applying a full-world compensation.

## Marker opacity and covered state

Since 5.20.0, `Marker` and `MarkerOptions` accept numbers as well as strings for `opacity` and `opacityWhenCovered`.

```js
new Marker({opacity: 1, opacityWhenCovered: 0.25});
```

A marker occluded by 3D terrain or a globe receives the `maplibregl-marker-covered` CSS class. Use that class for covered-state styling instead of inferring occlusion separately.
