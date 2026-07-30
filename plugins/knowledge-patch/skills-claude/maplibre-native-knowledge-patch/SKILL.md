---
name: maplibre-native-knowledge-patch
description: MapLibre Native
version: null
license: MIT
metadata:
  author: Nevaberry
---


# MapLibre Native

Use this skill when working on MapLibre Native core, Android, iOS, Node, or
Qt code; choosing a renderer backend; porting styles from MapLibre GL JS; or
building offline and headless-rendering workflows.

## Reference index

| Reference | Topics |
| --- | --- |
| [android.md](references/android.md) | Android upgrade breaks, renderer artifacts, camera and style features, offline regions, and ambient cache |
| [ios.md](references/ios.md) | iOS upgrades, observers, snapshots, camera controls, style APIs, distribution, networking, and Metal plugin layers |
| [desktop-bindings.md](references/desktop-bindings.md) | Node runtime support, rendering and request hooks, logging, and Qt 3 CMake, QML, deployment, and Android ABI setup |
| [style-and-interop.md](references/style-and-interop.md) | Native style support, expressions, runtime mutation, and GL JS interoperability boundaries |
| [architecture-and-rendering.md](references/architecture-and-rendering.md) | Repository boundaries, threading, tile preparation, drawables, shader and pass design, backend support, Linux builds, and render tests |

## Breaking changes and migrations

### Select the Android renderer artifact deliberately

`org.maplibre.gl:android-sdk` uses Vulkan on Android 13. Use the OpenGL ES
artifact when Vulkan is not intended:

```kotlin
implementation("org.maplibre.gl:android-sdk-opengl:13.4.0")
```

Vulkan surface snapshots require 13.3 or later, and Vulkan custom layers
require 13.4 or later. Recheck color-relief and hillshade ordering when moving
to Vulkan; 13.0.1 fixed those layers disappearing above fill layers.

### Raise Android's minimum SDK before upgrading

Android 12 requires `minSdk` 23 or later. From 12.1, a failed
`System.loadLibrary` throws instead of leaving the load failure silent, so
make native-loading failures part of startup error handling.

### Migrate synchronous GeoJSON configuration

Android 13 removed the short-lived synchronous setter variants. Configure the
source once and then use the normal update API:

```kotlin
GeoJsonOptions().withSynchronousUpdate(true)
```

iOS gained synchronous GeoJSON source updates separately; do not translate
the Android construction pattern directly to Darwin APIs.

### Retire legacy Android annotations

The core `Annotation` hierarchy, including `Marker`, `Polyline`, `Polygon`,
`MarkerOptions`, and `IconFactory`, is deprecated. Build new overlays with the
separate MapLibre Annotation Plugin.

### Require a supported Node runtime

The Node package no longer supports Node.js 16 because its packaging path
requires Node.js 18 or newer. Stable binding 6.4.1 supports Node.js 20, 22,
and 24; Node.js 26 belongs to the 6.5 prerelease line.

Linux and Windows Node builds use the drawable renderer; code that assumes
the removed legacy renderer must be updated.

### Treat platform releases as independent

Android, iOS, Node, and Qt have separate release streams. Do not infer core or
cross-platform feature parity from one platform's release number. Renderer
backends are also selected at build time and compiled with the wrapper and
core; they are not interchangeable runtime modules.

## High-use capabilities

### PMTiles and MLT

- Android supports PMTiles from 11.8.0; PMTiles joins the ambient cache from
  13.3.
- iOS supports `pmtiles://` sources from 6.10. From 6.14, their metadata is
  always interpreted with the XYZ tile scheme.
- Node supports PMTiles from 6.1.
- Android 12.1 and iOS 6.20 parse MLT vector-tile sources. Android 13.4 also
  accepts FastPFOR-encoded MLT tiles.

When a Node style uses a custom URL scheme, its constructor `request` hook
must resolve that scheme and return uncompressed resource bytes.

### Runtime feature state

Vector-tile feature-state updates are applied completely in iOS 6.15 and
later. Android exposes feature-state functionality from 13.4. Keep these
platform boundaries explicit when sharing interaction logic.

### Camera and viewport controls

- iOS supports maximum camera bounds, configurable north snapping, frustum
  offset, basic camera roll, and near-clipped projection access for custom
  layers.
- Android supports frustum offset from 12.2 and camera roll from 13.0.
- Android 13.1 changed pitched-map icon offsets by disabling icon scaling when
  offsets are present; visually recheck affected styles.

### Snapshot and headless rendering

Android snapshots can hide attribution from 12.0.1 and accept padding from
12.1. iOS can hide attribution and add snapshotter annotations from 6.21.
iOS 6.26.1 exposes a headless Metal texture for direct interoperability.

For desktop rendering, Node returns an asynchronous raw four-channel pixel
buffer, while `mbgl-render` writes a PNG from a style. Use `xvfb-run` when
Linux rendering has no X display.

### Style and source evolution

Color-relief layers and updated hillshade algorithms arrive in iOS 6.24 and
Android 13.0. Rounded fill extrusions arrive in Android 13.4.

Native styles are feature-specific rather than automatically equivalent to
GL JS. Check Android and iOS support separately for every root property,
source option, layer property, expression, and protocol. In particular:

- `centerAltitude`, root `roll`, and global-state functionality are
  unsupported on Android and iOS.
- Native pitch is limited to 0–60 degrees.
- A `glyphs` URL works, but omitting it for local fonts does not.
- iOS has typed vector, raster, raster-DEM, GeoJSON, and image sources, but
  not canvas or video sources.

### Assertions and coercions

`get` yields a generic value. Assert the concrete type required by the
consumer:

```json
["string", ["get", "feature_property"]]
```

Operators beginning with `to-` coerce and may include a fallback:

```json
["to-number", ["get", "feature_property"], 0]
```

Android and iOS builders ultimately feed the shared C++ expression evaluator,
but their typed property names and value wrappers remain platform-specific.

## Platform integration rules

### Wait for a loaded style

On Android, mutate the loaded `Style` proxy with typed sources, layers,
images, light, transitions, and ordered layer insertion. On iOS, use
`MLNStyle`, `MLNSource`, and `MLNStyleLayer`. Do not derive typed native
property names mechanically from style JSON names.

### Keep offline stores distinct

Android explicit offline regions and the ambient cache have different
lifecycle and eviction rules. Clearing ambient data preserves resources
needed by offline regions. Treat database packing, reset, invalidation, and
metadata changes as asynchronous storage work, not frame work.

After iOS imports an offline database, call `reloadPacks` before reading
`MLNOfflineStorage.sharedOfflineStorage.packs`.

### Respect observer roles

A Map View owns viewport and configuration but does not render by itself. Map
observers cover style, camera, idle, and render lifecycle changes; rendering
observers cover frame events, which may propagate to map observers.

On iOS, observer hooks are available from 6.12 and `sourceDidChange` from
6.13. Layer observers receive source-layer and source-ID changes from 6.28.

### Keep resource work asynchronous

Tile workers are actors. Parsing, layout, glyph/image dependencies, and
collision metadata are coalesced per geometry tile before prepared buckets
become backend resources. Do not introduce shared mutable cross-thread tile
state or assume every intermediate camera state is replayed.

## Choosing the next reference

- For an Android dependency or API migration, start with
  [android.md](references/android.md).
- For Cocoa names, Swift distribution, observers, delegates, or plugin
  layers, start with [ios.md](references/ios.md).
- For server-side rendering or Qt deployment, use
  [desktop-bindings.md](references/desktop-bindings.md).
- For style portability and expressions, use
  [style-and-interop.md](references/style-and-interop.md).
- For core threading, renderer internals, backend selection, or Linux render
  fixtures, use
  [architecture-and-rendering.md](references/architecture-and-rendering.md).
