# Camera, globe, and terrain

## Projection selection

Projection type can be provided as an expression as of 5.0.0. Vertical
Perspective is also available as a projection mode.

## Globe rendering

Globe mode, terrain on the globe, and realistic globe atmosphere are available
as of 5.0.0. Sky rendering is disabled on the globe and blends back in during
a transition to Mercator. Fog is disabled for the unsupported combination of
Terrain3D and globe.

## Camera orientation

The camera supports pitch beyond 90 degrees and a roll angle as of 5.0.0.
Drag rotation pivots around the center of the screen.

## Globe coordinates and queries

`queryRenderedFeatures` supports globe queries across the international date
line as of 5.4.0; callers do not need to split a crossing query.

On a globe map, `unproject` clamps points to the visible horizon rather than
returning a coordinate beyond the visible globe as of 5.4.0.

Marker drag coordinates on globe maps no longer contain an erroneous ±360°
longitude offset as of 5.4.0. Drag handlers should use the reported longitude
without compensating for a full-world wrap.

## Terrain elevation

`queryTerrainElevation` returns actual altitude as of 5.0.0. Update any
calculation that assumes its earlier numeric semantics.

## Camera fitting and zoom snapping

As of 5.20.0, `MapOptions.zoomSnap` applies to programmatic camera methods:

- `fitBounds` and `fitScreenCoordinates` snap downward to keep the requested
  bounds visible.
- `jumpTo`, `easeTo`, and `flyTo` snap to the nearest valid increment.
- In Vertical Perspective, `fitBounds` honors its `maxZoom` option.

## Terrain skirts

`MapOptions.terrainSkirtLength` controls terrain skirt length as of 6.0.0.
Applications with transparent map backgrounds can tune it to suppress visible
vertical artifacts at terrain edges.

```js
const map = new Map({
  container: 'map',
  terrainSkirtLength: desiredSkirtLength
});
```

## Light transitions

Style light-position transitions interpolate in spherical rather than
Cartesian coordinates in 6.0.0. The transition preserves radial distance, so
the rendered path can differ from the same transition under v5 behavior.

