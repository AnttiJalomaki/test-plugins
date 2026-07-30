# Style and custom rendering

## Data-driven line properties

`line-dasharray` supports data-driven styling as of 5.8.0. Use a literal
expression when returning an array value:

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

`line-cap`, `line-miter-limit`, and `line-round-limit` also accept data-driven
expressions as of 5.20.0.

```js
layout: {
  'line-cap': ['match', ['get', 'cap'], 'round', 'round', 'butt']
}
```

## Raster-derived resampling

Raster, hillshade, and color-relief layers expose resampling paint properties
as of 5.20.0. Raster layers can select nearest-neighbor sampling, whose
behavior was also corrected in that batch:

```js
paint: {'raster-resampling': 'nearest'}
```

## Whole-layer opacity

`fill-layer-opacity` and `line-layer-opacity` composite opacity uniformly
across an entire layer as of 6.0.0. `line-layer-opacity` avoids opacity
accumulation where line geometry overlaps. Transparency encoded in
`line-color` continues to stack.

```js
paint: {
  'line-layer-opacity': 0.5
}
```

## Typed property access

Layout- and paint-property getters and setters use each property's actual
TypeScript type in 6.0.0, replacing broad `string` and `any` signatures.
Callers that relied on loose types must provide valid property names and
correctly typed values.

## Shared shader source

Replace the legacy shader pragma with the MapLibre spelling when moving shared
shader code to v6 (migration-v5-v6):

```diff
-#pragma mapbox
+#pragma maplibre
```

## Custom-layer matrices and projections

Custom layers on Mercator maps receive non-translated matrices as of 5.0.0.
Update renderers that assumed the matrices contained the former translation.

Custom-layer argument objects expose `getProjectionData` in 6.0.0. Use it to
obtain current projection information through supported render arguments.

## Symbol offsets

Icon offsets are no longer scaled with the icon in 6.0.0. Adjust offsets in
symbol layouts that relied on the former coupling.

