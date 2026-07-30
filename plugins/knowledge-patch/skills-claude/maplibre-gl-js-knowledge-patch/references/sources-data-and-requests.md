# Sources, data, and requests

## Feature intersection and queries

`StyleLayer.queryIntersectsFeature` takes one
`QueryIntersectsFeatureParams` object instead of positional parameters as of
5.0.0. Wrap every former argument in the corresponding object field.

Globe-view `queryRenderedFeatures` handles queries that cross the
international date line as of 5.4.0. Applications no longer need to split
those queries.

## Cluster feature IDs

For clustered circle data configured with `promoteId`, an unclustered feature
receives its promoted ID and a clustered feature receives `cluster_id` as of
5.0.0. Do not rely on the earlier undefined ID for unclustered features.

## Vector source encodings

Vector sources support MapLibre Tiles by setting `encoding: 'mlt'` as of
5.12.0:

```js
map.addSource('mlt-data', {
  type: 'vector',
  tiles: ['https://example.com/tiles/{z}/{x}/{y}'],
  encoding: 'mlt'
});
```

## Tile level of detail and overscaling

Public tile level-of-detail control is available as of 5.4.0. Applications can
influence tile selection without reaching into internal behavior.

In 5.12.0, `MapOptions.experimentalZoomLevelsToOverscale` selects how many zoom
levels are sliced and how many are scaled during vector-tile overscaling. This
can improve high-zoom performance. A value of `4` or less can also avoid
Safari crashes in affected scenarios.

```js
const map = new Map({
  container: 'map',
  experimentalZoomLevelsToOverscale: 4
});
```

The option is promoted to `zoomLevelsToOverscale` for v6
(migration-v5-v6). It can change rendering and
`queryRenderedFeatures()` results. Set it explicitly to `undefined` to retain
the previous overscaling behavior.

```js
const map = new Map({
  container: 'map',
  zoomLevelsToOverscale: undefined
});
```

## GeoJSON updates

GeoJSON-VT-backed GeoJSON data supports updates, including diff updates, as of
5.20.0. A complete static tile index is no longer required for every change.

In 6.0.0, `GeoJSONSource.setData` accepts only its data argument. The
`waitForCompletion` parameter is removed, and the method no longer returns the
source instance. Do not pass the former second argument or chain a source
method from the result.

```js
source.setData(nextData);
```

Nested objects in GeoJSON feature properties are encoded and decoded back into
objects in 6.0.0. Their serialized form uses the `__$json__` prefix. This is a
breaking change for integrations that consumed the previous unsupported object
representation.

## Request transforms

`setTransformRequest` accepts an asynchronous callback as of 5.20.0.
`RequestParameters.referrerPolicy` controls the referrer policy for tile
requests.

```js
map.setTransformRequest(async (url) => ({
  url,
  referrerPolicy: 'no-referrer'
}));
```

## Missing style images

In v6, a `styleimagemissing` listener cannot satisfy the current image request
by calling `Map#addImage`. Register `Map#setMissingStyleImageResolver` instead
(migration-v5-v6). The resolver may be synchronous or asynchronous. An
asynchronous resolver must add the image before its promise settles.

```js
map.setMissingStyleImageResolver(async (id) => {
  const image = await generateImage(id);
  map.addImage(id, image);
});
```

The `styleimagemissing` event remains useful for observing images that remain
unresolved.

## URL hash parsing

Hash-based location control uses `URLSearchParams` parsing and normalization in
6.0.0. Encoded locations such as `#10%2F3.00%2F-1.00` are accepted, while a
bare `#foo` normalizes to `#foo=`.

## Terrain and source validation

In 6.0.0, `map.setTerrain` validates its terrain configuration, and
`raster-dem` sources passed to `map.addSource` are no longer exempt from
validation.

A custom source type registered with `addSourceType` no longer invalidates the
whole style merely because the style specification has no schema for that
custom type.

## Raster alpha data

`RasterTileSource#setPremultiplyAlpha(false)` preserves raw RGBA values when
the alpha channel carries data rather than opacity as of 6.0.0.

```js
map.getSource('raw-raster').setPremultiplyAlpha(false);
```

