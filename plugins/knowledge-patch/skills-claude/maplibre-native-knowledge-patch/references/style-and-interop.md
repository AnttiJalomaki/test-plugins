# Style and interoperability

## Check native support feature by feature

A valid version 8 style that works in MapLibre GL JS is not necessarily
feature-equivalent on Android or iOS (`native-style`). For every root
property, source option, layer property, and expression, check the distinct
Native Android and Native iOS support-table entries, minimum versions, and
linked unsupported issues.

Protocol support arrives through Native releases. Do not assume the GL JS
protocol-registration API exists on Native.

## Root and camera differences

`centerAltitude`, root `roll`, and root `state` or global-state functionality
are unsupported on both Native Android and Native iOS (`native-style`).
Native pitch supports only 0–60 degrees rather than the wider GL JS ranges.

## Fonts and glyphs

A `glyphs` URL works in Native, but omitting it to use local fonts is
unsupported on Android and iOS (`native-style`).

Basic root `font-faces` support starts in Android 11.13.0 and iOS 6.18.0,
although GL JS does not support that root property.

## iOS source types

The iOS SDK maps vector, raster, raster-DEM, GeoJSON, and image sources to
typed `MLN*Source` classes (`native-style`). Canvas and video sources are
unsupported on iOS.

## Expression types

Expression results include the generic `value` type and concrete types such
as string, number, color, object, array, collator, and formatted text
(`native-style`).

Because `get` returns `value`, add an assertion when its consumer needs a
concrete type. An incorrect assertion fails when the expression is evaluated:

```json
["string", ["get", "feature_property"]]
```

Names beginning with `to-` are coercions rather than assertions. They may
supply a fallback:

```json
["to-number", ["get", "feature_property"], 0]
```

## Native expression construction

Android and iOS expose platform-specific expression builders. The resulting
expression is parsed and evaluated by the shared C++ core (`native-style`).
For example, an Android layer property can use nested builders:

```java
fillLayer.setProperties(
    fillColor(interpolate(
        exponential(0.5f), zoom(),
        stop(1.0f, color(Color.RED)),
        stop(5.0f, color(Color.BLUE)),
        stop(10.0f, color(Color.GREEN))
    ))
);
```

## Mutate only a loaded style

Android mutates a loaded `org.maplibre.android.maps.Style` proxy with typed
sources, layers, images, light, transitions, and indexed or relative layer
placement (`native-style`).

iOS performs the corresponding operations through `MLNStyle`, `MLNSource`,
and `MLNStyleLayer`. Wait until the style has loaded before mutating it. Check
native typed property names instead of inferring them directly from JSON
names.

## Shared artifacts versus runtime APIs

MapLibre GL JS and Native share shaders, the style specification, and
render-test fixtures, but do not share their Map, Style, Layer, Glyph,
TileWorker, or rendering implementations (`native-js-interop`).

Compatible style JSON and assets can cross the boundary. Visual fixture
parity does not imply runtime API parity or complete feature parity.

## Porting from GL JS

There is no one-to-one GL JS-to-Native method map
(`native-js-interop`). Port the shareable styles and assets, then implement
application integration against the chosen Native SDK. Do not translate
browser calls mechanically.
