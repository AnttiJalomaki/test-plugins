# Styles, projections, and custom rendering

## Projections, globe, terrain, and atmosphere

Projection type can be provided as an expression since 5.0.0, and Vertical Perspective is available as a projection mode.

Globe rendering supports terrain and an option for realistic atmosphere. Sky rendering is disabled while on the globe and blends back in during the transition to Mercator. Fog is disabled for the unsupported combination of Terrain3D and globe.

In 6.0.0, `MapOptions.terrainSkirtLength` controls terrain skirt length. Adjust it when transparent map backgrounds reveal vertical artifacts at terrain edges.

```js
const map = new Map({
  container: 'map',
  terrainSkirtLength: desiredSkirtLength
});
```

## Data-driven line properties

`line-dasharray` supports data-driven styling as of 5.8.0. Use a `literal` expression for the selected array where needed:

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

Since 5.20.0, `line-cap`, `line-miter-limit`, and `line-round-limit` can also use data-driven expressions.

```js
layout: {
  'line-cap': ['match', ['get', 'cap'], 'round', 'round', 'butt']
}
```

## Whole-layer opacity

The 6.0.0 `fill-layer-opacity` and `line-layer-opacity` paint properties apply opacity uniformly while compositing an entire layer. `line-layer-opacity` prevents opacity accumulation where line geometry overlaps. Transparency in `line-color` retains the older stacking behavior, so choose based on the desired overlap result.

```js
paint: {'line-layer-opacity': 0.5}
```

## Raster-derived resampling

Raster, hillshade, and color-relief layers support resampling paint properties as of 5.20.0. Raster layers can select nearest-neighbor sampling, whose behavior was also corrected in that batch:

```js
paint: {'raster-resampling': 'nearest'}
```

## Typed style properties

In 6.0.0, TypeScript signatures for layout- and paint-property getters and setters use the actual type of each property. Replace misspelled or dynamically broad property names, and supply values matching the selected property's type instead of relying on former `string` and `any` signatures.

## Custom-layer matrices and projection data

Starting in 5.0.0, Mercator custom layers receive non-translated matrices. Remove rendering math that assumes the matrices contain the former translation.

In 6.0.0, the argument objects passed to custom layers expose `getProjectionData`. Use this supported entry point to obtain current projection data rather than reaching into map internals.

## Shader pragmas

During the v6 migration, shared shader code must replace the legacy Mapbox spelling with the MapLibre pragma:

```diff
-#pragma mapbox
+#pragma maplibre
```

## Light transitions

Style light-position transitions interpolate in spherical coordinates as of 6.0.0 rather than in Cartesian coordinates. The interpolation preserves radial distance, so the visible path can differ from an equivalent v5 transition.

## Symbol icon offsets

Icon offsets no longer scale with the icon in 6.0.0. Recalculate offsets in symbol layouts that relied on the previous coupling between icon scale and placement offset.
