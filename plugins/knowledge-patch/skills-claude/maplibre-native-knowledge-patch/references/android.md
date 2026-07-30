# Android

## Upgrade and renderer decisions

### PMTiles

MapLibre Android 11.8.0 supports PMTiles-backed map data
(`android-11.8.0`). Starting in 13.3, PMTiles sources also participate in the
ambient cache (`android-13.0.0`).

### Android 12 migration

MapLibre Android 12.0 raises the minimum supported Android SDK level from 21
to 23. Set the application's `minSdk` to at least 23 before upgrading
(`android-12.0.0`).

From 12.1, a failed `System.loadLibrary` throws an exception instead of
allowing native-library loading to fail silently. Handle the failure where
the application initializes the SDK.

### Vulkan-default Android 13 artifact

From Android 13.0, `org.maplibre.gl:android-sdk` selects Vulkan. Applications
that intend to stay on OpenGL ES must use
`org.maplibre.gl:android-sdk-opengl` instead (`android-13.0.0`):

```kotlin
implementation("org.maplibre.gl:android-sdk-opengl:13.4.0")
```

Vulkan surface snapshots arrive in 13.3, and Vulkan custom layers in 13.4.
Version 13.0.1 fixes color-relief and hillshade layers becoming invisible
above fill layers on the default Vulkan backend.

### Pitched-map icon offsets

Android 13.1 disables icon scaling when icon offsets are used on pitched maps.
Recheck styles combining pitch and icon offsets for visual changes
(`android-13.0.0`).

## Sources, tiles, and style features

### Synchronous GeoJSON updates

Android 12.3 introduced synchronous GeoJSON source updates for changes that
must be visible before execution continues (`android-12.0.0`).

Android 13.0 then replaced the individual synchronous setter methods. Enable
synchronous behavior through `GeoJsonOptions` and use the ordinary GeoJSON
update API (`android-sdk`):

```kotlin
GeoJsonOptions().withSynchronousUpdate(true)
```

### MLT tiles

Android 12.1 parses vector-tile sources in MLT format
(`android-12.0.0`). Android 13.4 additionally supports MLT tiles encoded with
FastPFOR (`android-13.0.0`).

### Feature state

Android 13.4 adds runtime per-feature state (`android-13.0.0`). Use it for
state-dependent styling or interaction without rewriting source data.

### Color relief and rounded extrusion

Android 13.0 adds Color-Relief Layer support and updated hillshade algorithms.
Android 13.4 adds the fill-extrusion property for rounded corners on extruded
buildings (`android-13.0.0`).

## Camera, viewport, and snapshots

Android 12.0.1 lets snapshots hide attribution. Android 12.1 adds padding to
`MapSnapshotter` (`android-12.0.0`).

Android 12.2 adds frustum offset, allowing rendering to omit screen edges
(`android-12.0.0`). Android 13.0 adds camera roll (`android-13.0.0`).

## Annotations

The core `Annotation` hierarchy has been deprecated since 7.0.0. This includes
`Marker`, `Polyline`, `Polygon`, `MarkerOptions`, and `IconFactory`. New
annotation work should use the separate MapLibre Annotation Plugin
(`android-sdk`).

## Explicit offline regions

`OfflineManager.getInstance(context)` manages persistent region definitions
with opaque application metadata (`android-sdk`). Its operations are
asynchronous, and callbacks arrive on the main thread.

A region observer reports progress and errors. `setDownloadState` pauses or
resumes fetching without making downloaded resources unavailable.
`invalidate`, `updateMetadata`, and `delete` respectively revalidate
resources, replace opaque metadata, and make resources eligible for eviction.

## Ambient cache

The ambient cache contains resources encountered during normal rendering and
is distinct from explicit offline regions (`android-sdk`):

- `setMaximumAmbientCacheSize` sets its limit in bytes.
- `invalidateAmbientCache` revalidates ambient resources.
- `clearAmbientCache` evicts ambient data but preserves resources required by
  offline regions.
- `putResourceWithUrl` prewarms it with response bytes and HTTP cache
  metadata.

## Offline database maintenance

`packDatabase` compacts the offline database.
`runPackDatabaseAutomatically` controls automatic compaction, and
`resetDatabase` deletes and reinitializes the store (`android-sdk`). These are
asynchronous storage operations and should not execute as part of frame
rendering.
