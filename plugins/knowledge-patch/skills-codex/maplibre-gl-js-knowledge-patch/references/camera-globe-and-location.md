# Camera, globe interactions, and location

## Orientation and rotation

The camera supports roll and pitch beyond 90 degrees starting in 5.0.0. Drag rotation pivots around the center of the screen. Audit camera constraints, controls, and assumptions that pitch stays below a horizon-facing orientation.

The 6.0.0 map event map includes roll lifecycle events, so typed applications can observe roll changes without adding an untyped extension.

## Terrain elevation

`queryTerrainElevation` returns actual altitude as of 5.0.0. Code written around the earlier numeric semantics must reinterpret or remove any compensating conversion.

`map.setTerrain` validates its configuration in 6.0.0. Handle invalid terrain input rather than relying on it to bypass validation.

## Globe queries and horizon behavior

Since 5.4.0, globe-view `queryRenderedFeatures` supports query regions that cross the international date line. Remove code that splits a query solely to work around wrapping at that boundary.

On a globe map, `unproject` clamps screen points to the visible horizon instead of returning a location beyond the globe's visible surface. Treat the returned coordinate as the horizon intersection.

## Camera fitting and zoom snapping

As of 5.20.0, `zoomSnap` applies to programmatic camera methods:

- `fitBounds` and `fitScreenCoordinates` snap downward so the requested bounds remain visible.
- `jumpTo`, `easeTo`, and `flyTo` snap to the nearest valid increment.
- In Vertical Perspective projection, `fitBounds` honors its `maxZoom` option.

Do not apply a second manual snap unless a distinct application policy requires it.

## URL hash parsing

Hash-based location control uses `URLSearchParams` parsing and normalization in 6.0.0. Encoded location strings such as `#10%2F3.00%2F-1.00` are accepted. A bare value such as `#foo` normalizes to `#foo=`; account for that normalization in routers, tests, and copied URLs.
