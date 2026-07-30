# Runtime and migration

## Packaging and imports

### ESM-only distribution

The v6 migration removes UMD and dedicated CSP bundles and distributes
`maplibre-gl.mjs` with `maplibre-gl-worker.mjs` (migration-v5-v6). Named npm
imports continue to work. Replace default imports with namespace or named
imports, and use `type="module"` for direct browser loading.

```ts
import * as maplibregl from 'maplibre-gl';
import {Map, setWorkerUrl} from 'maplibre-gl';
```

```html
<script type="module">
  import * as maplibregl from 'https://unpkg.com/maplibre-gl@^6.0.0/dist/maplibre-gl.mjs';
</script>
```

The unminified production build was already removed in 5.0.0. Consumers that
reference that artifact must select a distributed ESM build or produce the
needed debugging output through their own toolchain.

## Worker loading and CSP

Migration previews described direct browser ESM resolving its worker relative
to `import.meta.url`, with a same-origin Blob URL for a cross-origin CDN. Under
that preview behavior, CDN deployments needed `blob:` in `worker-src`, while a
self-hosted worker did not (migration-v5-v6):

```text
worker-src 'self' blob:;
img-src data: blob: 'self';
```

The final 6.0.0 build supersedes that preview path: it loads the worker as a
real module URL, and direct CDN use auto-loads the cross-origin worker while
preserving ESM semantics. It therefore does not need the CSP-specific bundle
or a `worker-src blob:` allowance.

Bundled applications must still call `setWorkerUrl()` once when their bundler
cannot reliably retain worker resolution through `import.meta.url`
(migration-v5-v6). Validate the emitted worker URL rather than assuming direct
browser behavior also applies to a bundler.

Scripts imported into a worker can communicate with the worker environment and
call `makeRequest` from the worker as of 5.20.0.

## Browser and graphics requirements

Published v6 code targets ES2022. Older browsers or build tools need to be
updated or handled by application-side transpilation (6.0.0).

WebGL 1 support is removed in 6.0.0; WebGL 2 is required. A failure to create
the WebGL context is emitted through the map's `error` event:

```js
map.on('error', handleMapError);
```

Legacy compatibility paths for IE11 and pre-2016 browsers were removed in
5.20.0 in favor of native browser APIs. Image requests always send
`Accept: image/webp`; the old Edge 18 detection workaround is gone.

## WebGL context construction

Since 5.0.0, WebGL options such as `antialias`, `preserveDrawingBuffer`, and
`failIfMajorPerformanceCaveat` belong to
`MapOptions.canvasContextAttributes`. `contextType` selects the WebGL version.
The v5 runtime could use WebGL 2 with a WebGL 1 fallback; v6 requires WebGL 2.

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

## Map and camera architecture

In 6.0.0, `Map` composes a `Camera` and forwards its public API rather than
inheriting from `Camera`. Code that checks the inheritance relationship or
uses internal `map.transform` state must move to public `Map` methods.
`transform.getMatrixForModel` is removed.

## Network failures

Fetch failures, including CORS, DNS, and malformed-URL failures, are delivered
as `AJAXError` instances through the map's `error` event since 5.0.0. Error
handlers can inspect the request details exposed by the error.

