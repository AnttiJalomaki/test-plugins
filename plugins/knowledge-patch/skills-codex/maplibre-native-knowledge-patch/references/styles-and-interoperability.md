# Styles, Sources, and Interoperability

## Check compatibility feature by feature

A valid version 8 style that works in MapLibre GL JS is not necessarily
feature-equivalent on Native. For every root property, source option, layer
property, and expression, check the separate Native Android and Native iOS
support-table entries, minimum versions, and linked unsupported issues.

Protocol support arrives through Native releases rather than the GL JS
protocol-registration API.

## Shared assets are not shared runtime APIs

MapLibre Native and GL JS share shaders, the style specification, and
render-test fixtures. They do not share Map, Style, Layer, Glyph, TileWorker, or
renderer implementations.

Compatible style JSON and assets can cross the boundary, but visual fixture
parity does not imply public API parity or complete feature parity. There is no
one-to-one browser-to-Native method map: port styles and assets, then implement
application integration against the target Native SDK.

## Root properties and camera limits

Native Android and iOS do not support:

- root `centerAltitude`;
- root `roll`;
- root `state` and global-state functionality.

Native pitch is limited to 0–60 degrees, narrower than the browser ranges.

## Glyphs and fonts

A `glyphs` URL works in Native. Omitting it to use local fonts is unsupported on
Android and iOS.

Conversely, basic root `font-faces` support starts in Android 11.13.0 and iOS
6.18.0 even though GL JS does not support that root property.

Core glyphs are 24-pixel signed-distance-field bitmaps in a texture atlas packed
in a protobuf container. Pixels inside an outline use values 192–255; outside
pixels use 0–191. The representation supports GPU resizing, rotation, and halo
rendering from the shared atlas.

## Source support

The iOS SDK maps vector, raster, raster-DEM, GeoJSON, and image sources to typed
`MLN*Source` classes. Canvas and video sources are unsupported on iOS.

PMTiles is available on Android, iOS, and Node. iOS addresses it through
`pmtiles://` and treats PMTiles metadata as XYZ from 6.14. MLT parsing arrives
in Android 12.1 and iOS 6.20; Android 13.4 supports FastPFOR-encoded MLT tiles.

## Assertions and coercions

Expression results include generic `value` and concrete types such as string,
number, color, object, array, collator, and formatted text. Because `get`
returns `value`, assert a concrete type when required by the consuming
expression:

```json
["string", ["get", "feature_property"]]
```

An assertion fails at evaluation time if the value has the wrong type.

Operators whose names begin with `to-` are coercions rather than assertions and
may include a fallback:

```json
["to-number", ["get", "feature_property"], 0]
```

## Native expression construction

Android and iOS provide platform-specific expression builders, but the shared
C++ core parses and evaluates the resulting expression. Do not assume that the
surface syntax means a different evaluation engine.

On iOS, layout and paint values use `NSExpression` with Cocoa values such as
`UIColor`, `CGVector`, and `UIEdgeInsets`; the older style-function API is
unsupported. Layer filters use `NSPredicate`.

## Loaded-style mutation

Wait for style loading to complete before runtime mutation.

Android mutates `org.maplibre.android.maps.Style` with typed sources, layers,
images, light, transitions, and indexed or relative layer placement. iOS uses
`MLNStyle`, `MLNSource`, and `MLNStyleLayer`.

Always check the target SDK's typed property name. Do not infer it directly from
the JSON spelling; for example, iOS uses `lineDashPattern`,
`rasterHueRotation`, `iconImageName`, `iconScale`, `text`, `textFontNames`, and
`textFontSize`.
