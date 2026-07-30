# Packaging and runtime

## Distribution and imports

The unminified production build was removed in 5.0.0. Select another distributed artifact rather than retaining a direct path to that file.

The ESM migration (migration-v5-v6) removes the UMD and dedicated CSP bundles. The package distributes `maplibre-gl.mjs` and `maplibre-gl-worker.mjs`:

- Named npm imports continue to work.
- Replace a default import with `import * as maplibregl from 'maplibre-gl'` or named imports such as `import {Map, setWorkerUrl} from 'maplibre-gl'`.
- Direct browser loading must use `<script type="module">` and import the `.mjs` entry.

```html
<script type="module">
  import * as maplibregl from 'https://unpkg.com/maplibre-gl@^6.0.0/dist/maplibre-gl.mjs';
</script>
```

## Worker URLs and CSP

The migration design initially described direct browser ESM resolving its worker relative to `import.meta.url` and using a same-origin Blob URL for a cross-origin CDN. Under that design, CDN pages needed `worker-src 'self' blob:` and `img-src data: blob: 'self'`, while a self-hosted worker did not need the `worker-src blob:` allowance.

The final 6.0.0 ESM build supersedes that transitional behavior: it loads a real module worker URL and automatically loads a cross-origin CDN worker while preserving ESM semantics. Final CDN usage therefore does not require the CSP-specific bundle or a `worker-src blob:` allowance.

Bundlers still cannot reliably resolve the worker with `import.meta.url`. Call `setWorkerUrl()` once in a bundled application. For direct browser ESM, allow automatic worker discovery and do not add the call merely out of habit.

## JavaScript, browsers, and WebGL

Published 6.0.0 code targets ES2022. Update old browsers and tooling, or configure application-side transpilation.

WebGL 1 support is removed in 6.0.0; WebGL 2 is required. WebGL-unavailable failures are emitted on the map's `error` event, so install an error listener if the application must report or recover from initialization failure.

At 5.0.0, WebGL configuration moved from top-level map options to `MapOptions.canvasContextAttributes`. This includes `antialias`, `preserveDrawingBuffer`, and `failIfMajorPerformanceCaveat`. `contextType` chooses the context version. The v5 runtime could request WebGL 2 with a WebGL 1 fallback; the final v6 runtime cannot fall back to WebGL 1.

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

Legacy compatibility paths for IE11 and pre-2016 browsers were removed in 5.20.0 in favor of native APIs. Image requests now always send `Accept: image/webp`; do not retain Edge 18 detection logic to decide that header.

## `Map` and `Camera` architecture

As of 6.0.0, `Map` composes a `Camera` and forwards its public API rather than inheriting from `Camera`. Code must not rely on `map instanceof Camera`, subclass inheritance details, or the internal `map.transform` object. Use public map APIs.

`transform.getMatrixForModel` is removed. Custom rendering code should use the supported render arguments, including `getProjectionData` where projection details are needed.
