---
name: maplibre-gl-js-knowledge-patch
description: MapLibre GL JS
version: 6.0.0
license: MIT
metadata:
  author: Nevaberry
---


# MapLibre GL JS Knowledge Patch

Use this skill when migrating MapLibre GL JS applications, updating map
configuration, or working with events, sources, projections, terrain, styles,
workers, and custom rendering.

## Reference index

| Reference | Topics |
|---|---|
| [Runtime and migration](references/runtime-and-migration.md) | ESM packaging, workers and CSP, browser and WebGL requirements, map construction, runtime errors |
| [Events and controls](references/events-and-controls.md) | Subscriptions, event typing, style loads, geolocation, box zoom, reduced motion, popups, markers |
| [Sources, data, and requests](references/sources-data-and-requests.md) | Queries, vector and GeoJSON sources, overscaling, request transforms, missing images, validation, raster alpha |
| [Camera, globe, and terrain](references/camera-globe-and-terrain.md) | Projections, globe queries, camera orientation, fitting, elevation, terrain skirts, lighting |
| [Style and custom rendering](references/style-and-custom-rendering.md) | Data-driven properties, resampling, layer opacity, shader pragmas, custom-layer matrices and projection data |

## Use this patch

1. Read the installed `maplibre-gl` version before changing code.
2. For an upgrade to v6, start with Runtime and migration, then review Events
   and controls because packaging and event types both changed substantially.
3. Open the topic reference that owns the API being changed. Several familiar
   APIs still exist but now have different return types, parameter shapes, or
   rendering semantics.
4. Prefer public `Map` APIs and supported render arguments over internal
   `transform` state.
5. Test worker loading, CSP, WebGL availability, feature queries, and custom
   rendering in the same deployment topology used in production.

## Critical v6 migration

### Import the ESM distribution

The package is ESM-only. UMD and dedicated CSP bundles are gone. Named npm
imports remain valid, while a former default import must become a namespace or
named import.

```ts
import * as maplibregl from 'maplibre-gl';
import {Map, setWorkerUrl} from 'maplibre-gl';
```

Direct browser loading requires a module script and the `.mjs` build.

```html
<script type="module">
  import {Map} from 'https://unpkg.com/maplibre-gl@^6.0.0/dist/maplibre-gl.mjs';
</script>
```

### Configure workers for the actual deployment

The final ESM build loads its worker as a real module URL. Direct CDN loading
auto-loads the cross-origin worker without the former CSP-specific bundle or a
`worker-src blob:` exception. In a bundled application, call `setWorkerUrl()`
once when the bundler cannot preserve worker URL resolution.

Do not copy CSP advice from preview migration builds without checking the final
worker path. Self-hosted and direct-CDN deployments have different origin
constraints.

### Meet the runtime floor

Published JavaScript targets ES2022 and requires WebGL 2. Update older tooling
or transpile in the application where necessary. WebGL initialization failures
arrive on the map's `error` event.

```js
map.on('error', handleMapError);
```

Configure context creation under `canvasContextAttributes`, not with former
top-level map options.

```js
const map = new Map({
  container: 'map',
  canvasContextAttributes: {
    antialias: true,
    preserveDrawingBuffer: true,
    contextType: 'webgl2'
  }
});
```

### Stop depending on `Map` inheritance

`Map` composes and forwards a `Camera`; it no longer extends `Camera`. Replace
inheritance checks and access to `map.transform` with public map methods.
`transform.getMatrixForModel` is removed.

### Update event code

Listeners return a `Subscription`, so fluent listener registration no longer
works. Register separately and retain subscriptions that must be removed.

```js
const moveSubscription = map.on('move', onMove);
map.on('zoom', onZoom);
moveSubscription.unsubscribe();
```

Event objects are classes, but identify them through `event.type`, not
`instanceof`. `Evented` is abstract and generic over an event map. Replace
`MapDataEvent` with `MapSourceDataEvent | MapStyleDataEvent`, and rename
`MapLibreZoomEvent` uses to `MapBoxZoomEvent`.

### Update source mutation code

`GeoJSONSource.setData(data)` no longer accepts `waitForCompletion` and no
longer returns the source. Do not pass a second argument or chain from it.

```js
source.setData(nextData);
```

Nested objects in GeoJSON feature properties now round-trip as objects through
an internal `__$json__`-prefixed representation. Recheck integrations that
depended on the former unsupported serialized shape.

### Resolve missing style images before completion

`styleimagemissing` can observe unresolved images but cannot satisfy the
current request by calling `addImage()`. Install a synchronous or asynchronous
resolver instead. An asynchronous resolver must add the image before its
promise settles.

```js
map.setMissingStyleImageResolver(async (id) => {
  const image = await generateImage(id);
  map.addImage(id, image);
});
```

### Review vector-tile overscaling

The v5 experimental tile-slicing option is named `zoomLevelsToOverscale` in
v6. It can alter rendering and `queryRenderedFeatures()` results. Set it to
`undefined` explicitly when preserving the former overscaling behavior.

```js
const map = new Map({
  container: 'map',
  zoomLevelsToOverscale: undefined
});
```

### Update shared shaders

Custom shader source must use the MapLibre pragma spelling.

```glsl
#pragma maplibre
```

## High-use API changes

### Typed events and style properties

TypeScript applications may extend `MapEventType` with declaration merging.
Layout and paint getters and setters now enforce the actual type of each
property, so correct property names and values are required.

```ts
declare module 'maplibre-gl' {
  interface MapEventType {
    'app:ready': {type: 'app:ready'; payload: string};
  }
}
```

### Style lifecycle

`setStyle()` emits `style.load` when it applies style JSON as a diff as well as
when it fully reloads the style.

```js
map.once('style.load', onStyleLoad);
map.setStyle(nextStyle);
```

### Async request transformation

Request transforms may be asynchronous. Use
`RequestParameters.referrerPolicy` when tile requests need an explicit
referrer policy.

```js
map.setTransformRequest(async (url) => ({
  url,
  referrerPolicy: 'no-referrer'
}));
```

### Data-driven lines

`line-dasharray`, `line-cap`, `line-miter-limit`, and `line-round-limit`
support feature expressions. Use literal arrays where an expression must
return a dash pattern.

```js
paint: {
  'line-dasharray': [
    'match',
    ['get', 'kind'],
    'rail', ['literal', [2, 2]],
    ['literal', [1, 0]]
  ]
}
```

### Whole-layer opacity

Use `fill-layer-opacity` or `line-layer-opacity` to composite opacity uniformly
across a layer. `line-layer-opacity` avoids accumulation at overlapping line
geometry; alpha embedded in `line-color` still stacks.

### Camera fitting and snapping

`fitBounds` and `fitScreenCoordinates` snap zoom downward so requested bounds
remain visible. `jumpTo`, `easeTo`, and `flyTo` snap to the nearest configured
increment. Vertical Perspective `fitBounds` also honors `maxZoom`.

### Accessible motion and UI placement

Set `MapOptions.reduceMotion` for map-level reduced-motion behavior. Use
`Popup({padding})` to keep automatic placement away from container edges.
Markers accept numeric opacity values and receive
`maplibregl-marker-covered` when hidden by terrain or the globe.

