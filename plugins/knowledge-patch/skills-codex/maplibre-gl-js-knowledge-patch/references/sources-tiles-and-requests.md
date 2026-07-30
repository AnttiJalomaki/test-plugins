# Sources, tiles, and requests

## Feature queries

At 5.0.0, `StyleLayer.queryIntersectsFeature` changed from positional parameters to one object conforming to `QueryIntersectsFeatureParams`. Wrap every former argument in that parameter object.

On globe maps, `queryRenderedFeatures` handles query regions that cross the international date line as of 5.4.0. Do not split those regions or apply the former workaround.

## Vector source encodings and feature IDs

MapLibre Tiles are accepted by vector sources since 5.12.0. Set `encoding: 'mlt'`:

```js
map.addSource('mlt-data', {
  type: 'vector',
  tiles: ['https://example.com/tiles/{z}/{x}/{y}'],
  encoding: 'mlt'
});
```

For clustered circle data with `promoteId`, the 5.0.0 ID rules are:

- An unclustered feature gets its promoted ID instead of `undefined`.
- A clustered feature gets `cluster_id`.

## Tile level of detail and overscaling

Public tile level-of-detail control became available in 5.4.0, allowing an application to influence tile selection through supported API rather than internal state.

`MapOptions.experimentalZoomLevelsToOverscale` was added in 5.12.0. It controls how many vector-tile zoom levels are sliced and how many are scaled during overscaling. Tuning it can improve high-zoom performance; a value of `4` or less can prevent Safari crashes in affected cases.

The v6 migration promotes the option to `zoomLevelsToOverscale`. It can change both rendering and `queryRenderedFeatures()` output. To preserve the earlier overscaling behavior, explicitly set it to `undefined`:

```js
const map = new Map({
  container: 'map',
  zoomLevelsToOverscale: undefined
});
```

## GeoJSON updates and property encoding

GeoJSON-VT-backed GeoJSON data supports updates, including diff updates, as of 5.20.0. Rebuilding an entirely static tile index is no longer required for every change.

In 6.0.0, `GeoJSONSource.setData` accepts only its data argument. Remove `waitForCompletion`, and do not chain from the result because the method no longer returns the source instance.

```js
source.setData(nextData);
```

Nested objects in GeoJSON feature properties are now encoded and parsed back as objects. The serialized representation uses the `__$json__` prefix. This is intentionally different from the former unsupported representation, so audit persistence or interop code that inspects serialized properties.

## Request customization and worker requests

Since 5.20.0, `setTransformRequest` accepts an async callback. `RequestParameters.referrerPolicy` controls the browser referrer policy for tile requests.

```js
map.setTransformRequest(async (url) => ({
  url,
  referrerPolicy: 'no-referrer'
}));
```

Scripts imported into workers can communicate with the worker environment and invoke `makeRequest` from inside a worker.

Image requests always send `Accept: image/webp`; Edge 18 detection is no longer used to choose that behavior.

## Validation

In 6.0.0, `map.setTerrain` validates its terrain configuration. A `raster-dem` source passed to `map.addSource` is also validated rather than skipped.

Custom sources are treated differently: a source type registered with `addSourceType` does not invalidate the whole style merely because the style specification lacks a schema for that custom type.

## Raster alpha data

Use `RasterTileSource#setPremultiplyAlpha(false)` when a raster tile's alpha channel contains data rather than opacity. Since 6.0.0 this preserves the raw RGBA values.

```js
map.getSource('raw-raster').setPremultiplyAlpha(false);
```
