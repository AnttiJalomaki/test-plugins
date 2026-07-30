---
name: maplibre-gl-js-knowledge-patch
description: MapLibre GL JS
version: 6.0.0
license: MIT
metadata:
  author: Nevaberry
---


# MapLibre GL JS Knowledge Patch

Use this skill when implementing, migrating, or reviewing MapLibre GL JS code. Start with the breaking-change checks below, then open the topic reference that matches the work.

## Index

| Reference | Topics |
|---|---|
| [Packaging and runtime](references/packaging-and-runtime.md) | ESM migration, worker loading, CSP, browser and WebGL requirements, build artifacts, `Map` composition |
| [Events, types, and errors](references/events-types-and-errors.md) | Subscriptions, event discrimination, event maps, request errors, `style.load`, missing images |
| [Sources, tiles, and requests](references/sources-tiles-and-requests.md) | Feature queries, MLT, overscaling, GeoJSON updates, request transforms, validation, raster alpha |
| [Styles, projections, and custom rendering](references/styles-projections-and-rendering.md) | Globe, expressions, layer properties, raster resampling, shaders, custom layers, light and icon behavior |
| [Camera, globe interactions, and location](references/camera-globe-and-location.md) | Pitch and roll, terrain elevation, horizon behavior, camera snapping, hash parsing |
| [Controls, markers, and UI](references/controls-markers-and-ui.md) | Geolocation, reduced motion, popup padding, box zoom, marker dragging and covered-state styling |

## Breaking-change triage

### Convert applications to ESM

The package exposes ESM files rather than UMD or dedicated CSP bundles. Keep named npm imports, and replace default imports with namespace or named imports:

```ts
import * as maplibregl from 'maplibre-gl';
// or
import {Map, setWorkerUrl} from 'maplibre-gl';
```

For direct browser use, load `dist/maplibre-gl.mjs` from a module script. Do not look for the removed unminified production, UMD, or CSP artifacts.

### Configure workers for the actual loading environment

The final ESM browser build resolves the worker as a real module URL and can auto-load a cross-origin CDN worker without a Blob-based CSP exception. Bundlers cannot reliably infer the worker URL, so call `setWorkerUrl()` once in bundled applications.

Do not carry forward the transitional migration assumption that CDN loading needs `worker-src blob:`; self-hosted and final module-worker deployments should express only the origins they actually use. See the packaging reference for the distinction.

### Enforce the runtime requirements

Published JavaScript targets ES2022 and rendering requires WebGL 2. Update browsers and build tooling or transpile at the application boundary. Listen for the map's `error` event because WebGL-unavailable failures arrive there.

```js
map.on('error', handleMapError);
```

### Stop relying on `Map extends Camera`

`Map` composes a `Camera` and forwards its public methods. Remove inheritance checks and internal `map.transform` access, and replace removed `transform.getMatrixForModel` use with supported map or custom-layer render APIs.

### Update WebGL construction options

Move top-level context settings under `canvasContextAttributes`. Use its `contextType` when selecting a context; current releases require WebGL 2.

```js
const map = new Map({
  container: 'map',
  canvasContextAttributes: {
    antialias: true,
    preserveDrawingBuffer: true,
    failIfMajorPerformanceCaveat: true,
    contextType: 'webgl2'
  }
});
```

### Treat event registration as a subscription

`Evented.on()` returns a `Subscription`, not the evented object. Split fluent listener chains and retain the subscription when it must be removed.

```js
const moveSubscription = map.on('move', onMove);
map.on('zoom', onZoom);
moveSubscription.unsubscribe();
```

Events are classes, but consumer code should discriminate them with `event.type`, not `instanceof`.

### Migrate event types

`Evented` is abstract and generic over an event map. Sources use `SourceEventType`; map events include roll lifecycle and typed `style.load` events. Replace `MapDataEvent` with `MapSourceDataEvent | MapStyleDataEvent`, and rename `MapLibreZoomEvent` to `MapBoxZoomEvent`.

Custom event declarations can merge into the `MapEventType` interface:

```ts
declare module 'maplibre-gl' {
  interface MapEventType {
    'app:ready': {type: 'app:ready'; payload: string};
  }
}
```

### Resolve missing style images through the resolver

A `styleimagemissing` handler cannot satisfy the request currently being processed by calling `addImage`. Install `setMissingStyleImageResolver`; it may return synchronously or asynchronously, but an async resolver must add the image before its promise settles.

```js
map.setMissingStyleImageResolver(async (id) => {
  const image = await generateImage(id);
  map.addImage(id, image);
});
```

Keep `styleimagemissing` only for observing images that remain unresolved.

### Update `GeoJSONSource.setData`

Pass only the data. The `waitForCompletion` argument and fluent return value are gone.

```js
source.setData(nextData);
```

Do not chain another source method from this call. Nested objects in feature properties now round-trip as objects and use the reserved serialized prefix `__$json__`; audit code that relied on the former unsupported representation.

### Use the promoted overscaling option deliberately

Rename `experimentalZoomLevelsToOverscale` to `zoomLevelsToOverscale`. The setting can alter rendering and `queryRenderedFeatures()` results. Set it explicitly to `undefined` when retaining the earlier overscaling behavior matters.

```js
const map = new Map({
  container: 'map',
  zoomLevelsToOverscale: undefined
});
```

### Pass query-intersection arguments as an object

`StyleLayer.queryIntersectsFeature` takes one `QueryIntersectsFeatureParams` object rather than the former positional arguments.

### Use exact style-property types

Layout and paint getters and setters now encode each property's real TypeScript type. Fix property names and values that previously compiled only because the signatures accepted broad `string` or `any` values.

### Update custom rendering code

Mercator custom layers receive non-translated matrices. Remove assumptions based on the former translated matrices and obtain current projection information from `getProjectionData` on the custom-layer argument object. Shared shaders must use the MapLibre pragma:

```glsl
#pragma maplibre
```

## High-use capabilities

### Await style diffs consistently

`setStyle()` emits `style.load` even when style JSON is applied as a diff.

```js
map.once('style.load', onStyleLoad);
map.setStyle(nextStyle);
```

### Configure async requests

`setTransformRequest` may be async, and returned `RequestParameters` can set `referrerPolicy`.

```js
map.setTransformRequest(async (url) => ({
  url,
  referrerPolicy: 'no-referrer'
}));
```

Imported worker scripts can communicate with their worker environment and call `makeRequest` from within a worker.

### Add MLT vector sources

Declare `encoding: 'mlt'` on a vector source that serves MapLibre Tiles.

```js
map.addSource('mlt-data', {
  type: 'vector',
  tiles: ['https://example.com/tiles/{z}/{x}/{y}'],
  encoding: 'mlt'
});
```

### Prefer whole-layer opacity when overlap must not accumulate

Use `fill-layer-opacity` or `line-layer-opacity` to composite opacity uniformly for the layer. `line-layer-opacity` avoids darkening where line geometry overlaps; alpha inside `line-color` still accumulates.

```js
paint: {'line-layer-opacity': 0.5}
```

### Make line geometry data-driven

Expressions can drive `line-dasharray`, `line-cap`, `line-miter-limit`, and `line-round-limit`. Wrap array outputs in `literal` where the expression grammar requires a literal array.

### Account for camera and globe semantics

Camera orientation supports roll and pitch beyond 90 degrees. Globe queries can cross the international date line, globe `unproject` clamps to the visible horizon, and marker drag longitudes no longer need a manual ±360° correction.

`fitBounds` and `fitScreenCoordinates` round zoom downward so requested bounds stay visible. `jumpTo`, `easeTo`, and `flyTo` round to the nearest `zoomSnap` increment. Vertical Perspective `fitBounds` honors `maxZoom`.

### Configure accessibility and overlays

Use `MapOptions.reduceMotion` for map-level reduced-motion behavior, `Popup({padding})` to keep automatic placement away from container edges, and numeric marker opacity values when convenient. Covered markers receive the `maplibregl-marker-covered` class.

## Review checklist

- Confirm all package loading uses ESM and bundled builds set a worker URL.
- Confirm the runtime can execute ES2022 and create a WebGL 2 context.
- Search for listener chaining, event `instanceof`, default package imports, and internal `map.transform` access.
- Search for two-argument or chained `setData` calls and positional `queryIntersectsFeature` calls.
- Check missing-image handlers, overscaling option names, shader pragmas, and custom-layer matrix assumptions.
- Compile TypeScript to expose stricter event and style-property signatures.
- Exercise globe horizon, date-line, marker-drag, terrain, and camera-snapping behavior where used.
- Review source validation, nested GeoJSON properties, raster alpha, request transforms, and worker requests.
- Inspect symbols and line opacity where placement, overlap, or transitions must match earlier rendering.
