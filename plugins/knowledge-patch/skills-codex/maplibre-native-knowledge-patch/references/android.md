# Android SDK

## Upgrade and renderer decisions

### Platform floor and library loading

At the `android-12.0.0` upgrade boundary, the minimum supported Android SDK
rises from 21 to 23. Set the application's `minSdk` to at least 23 before
upgrading.

From Android 12.1, a failed `System.loadLibrary` call throws an exception rather
than allowing native-library loading to fail silently. Handle or surface that
failure at initialization.

### Vulkan-default published artifact

At the `android-13.0.0` boundary, `org.maplibre.gl:android-sdk` switches to the
Vulkan renderer. Applications that need OpenGL ES must select
`org.maplibre.gl:android-sdk-opengl`:

```kotlin
implementation("org.maplibre.gl:android-sdk-opengl:13.4.0")
```

Vulkan surface snapshots arrive in 13.3 and Vulkan custom layers in 13.4.
Source builds separately expose `opengl` and `vulkan` Gradle flavors; do not
confuse the checkout's OpenGL broad-compatibility default with the Android 13
published artifact default.

## Sources and runtime data

### PMTiles and ambient caching

The `android-11.8.0` line adds PMTiles-backed map data. Starting in 13.3,
PMTiles sources also participate in the ambient cache.

### MLT tiles

Android 12.1 parses vector-tile sources in MLT format. Android 13.4 extends this
to MLT tiles using FastPFOR encoding.

### Synchronous GeoJSON updates

Android 12.3 introduces synchronous GeoJSON updates for cases where a source
change must be applied before execution continues. Android 13.0 replaces the
short-lived individual synchronous setter methods: construct the source with
`GeoJsonOptions.withSynchronousUpdate(true)`, then call the normal GeoJSON
update API.

```kotlin
GeoJsonOptions().withSynchronousUpdate(true)
```

### Feature state

Android 13.4 adds public feature-state functionality for runtime per-feature
state.

## Camera, snapshots, and style rendering

### Snapshotter controls

Android 12.0.1 allows snapshots to hide attribution. Android 12.1 adds padding
support to `MapSnapshotter`.

### Frustum offset and camera roll

Android 12.2 adds a frustum offset, allowing the renderer to omit screen edges.
Android 13.0 adds camera roll.

### Color relief and hillshade

Android 13.0 adds Color-Relief Layer support and updates hillshade algorithms.
Android 13.0.1 fixes color-relief and hillshade layers becoming invisible above
fill layers when using Vulkan, the default backend for the Android 13 artifact.

### Pitched icon offsets

Android 13.1 changes icon-offset behavior on pitched maps: icons no longer
scale when offsets are used. Recheck styles that combine pitch and icon offsets
for visual regressions.

### Rounded fill extrusions

Android 13.4 adds a fill-extrusion style property for rounded corners on
extruded buildings.

## Runtime style API

Android exposes platform expression builders, but the resulting expressions are
parsed and evaluated by the shared C++ core. Nested builders can construct a
layer property:

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

A loaded `org.maplibre.android.maps.Style` proxy supports typed sources, layers,
images, light, transitions, and indexed or relative layer placement. Wait for
the style to load before mutating it.

## Annotations

The core `Annotation` hierarchy has been deprecated since 7.0.0. This includes
`Marker`, `Polyline`, `Polygon`, `MarkerOptions`, and `IconFactory`. Use the
separate MapLibre Annotation Plugin for new overlay and annotation work.

## Offline regions

`OfflineManager.getInstance(context)` manages persistent region definitions
with opaque application metadata. Operations are asynchronous and callbacks
arrive on the main thread.

A region observer reports progress and errors. `setDownloadState` pauses or
resumes fetching without making downloaded resources unavailable.
`invalidate`, `updateMetadata`, and `delete` respectively revalidate resources,
replace metadata, or make resources eligible for eviction.

## Ambient cache

The ambient cache stores resources encountered during normal rendering and is
separate from explicit offline regions:

- `setMaximumAmbientCacheSize` sets the byte limit.
- `invalidateAmbientCache` revalidates ambient resources.
- `clearAmbientCache` evicts ambient data while retaining resources required by
  offline regions.
- `putResourceWithUrl` prewarms the cache with response bytes and HTTP cache
  metadata.

## Offline database maintenance

`packDatabase` compacts the offline database.
`runPackDatabaseAutomatically` controls automatic compaction. `resetDatabase`
deletes and reinitializes the store. These operations are asynchronous storage
work and should not run as part of frame rendering.
