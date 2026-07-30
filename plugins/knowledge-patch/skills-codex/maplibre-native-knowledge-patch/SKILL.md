---
name: maplibre-native-knowledge-patch
description: MapLibre Native
version: null
license: MIT
metadata:
  author: Nevaberry
---


# MapLibre Native Knowledge Patch

Use this skill when implementing, upgrading, building, or debugging MapLibre
Native applications and bindings. Start by identifying the target platform and
its SDK or binding version: Android, iOS or Darwin, Node, and Qt have independent
release streams, and there is no single public C++ core version that maps them
all.

Prefer the project's dependency manifest, selected renderer artifact, source
build flavor, code, and tests over general assumptions. Treat style and asset
compatibility separately from public API compatibility.

## Reference index

| Reference | Topics |
| --- | --- |
| [references/android.md](references/android.md) | Android upgrades, renderer artifacts, cameras, styles, offline storage, and caching |
| [references/ios-and-darwin.md](references/ios-and-darwin.md) | iOS upgrades, style APIs, observers, networking, snapshots, Metal plugins, and offline packs |
| [references/node-and-qt.md](references/node-and-qt.md) | Node rendering and resource hooks; Qt 3 CMake, QML, and deployment |
| [references/styles-and-interoperability.md](references/styles-and-interoperability.md) | Native style support, sources, expressions, runtime mutation, and GL JS interoperability |
| [references/architecture-and-rendering.md](references/architecture-and-rendering.md) | Core ownership, actors, tiles, drawables, shaders, passes, and backend support |
| [references/linux-build-and-testing.md](references/linux-build-and-testing.md) | Linux OpenGL builds, headless image rendering, and render fixtures |

## Breaking changes first

### Android renderer selection

The standard Android 13 artifact uses Vulkan:

```kotlin
implementation("org.maplibre.gl:android-sdk:13.4.0")
```

To remain on OpenGL ES, select the explicit artifact:

```kotlin
implementation("org.maplibre.gl:android-sdk-opengl:13.4.0")
```

Do not infer the packaged artifact default from the source checkout's
broad-compatibility flavor default. Vulkan surface snapshots require 13.3 and
Vulkan custom layers require 13.4.

### Android platform and loading requirements

Android 12 raises `minSdk` from 21 to 23. Update the application manifest or
Gradle configuration before upgrading. From 12.1, a failed
`System.loadLibrary` call throws instead of failing silently; keep initialization
paths ready to surface the exception.

### Node runtime requirements

Node.js 16 is unsupported from the Node 6.1 line because the package again uses
`@mapbox/node-pre-gyp`, which requires Node.js 18 or newer. For the stable 6.4.1
binding, use Node.js 20, 22, or 24. Node.js 26 support belongs to the 6.5
prerelease line.

Linux and Windows Node builds use the drawable renderer; the legacy renderer is
removed.

### Android annotation and GeoJSON migrations

The core `Annotation` hierarchy, including `Marker`, `Polyline`, `Polygon`,
`MarkerOptions`, and `IconFactory`, has been deprecated since 7.0.0. Use the
separate MapLibre Annotation Plugin for new annotation work.

Android 13 replaces the short-lived individual synchronous GeoJSON setters.
Enable synchronous updates in the source options, then use the normal update
API:

```kotlin
GeoJsonOptions().withSynchronousUpdate(true)
```

### Qt 3 migration

Qt 3 moves APIs into `QMapLibre`, splits Core, Location, and Widgets
installations, and exposes `QMapLibre::*` CMake targets. QML uses
`import MapLibre 3.0`. Deploy the matching library and plugin trees; linking a
library alone is not sufficient.

## High-value data-source changes

### PMTiles

PMTiles-backed data is supported by Android, iOS, and Node. iOS uses
`pmtiles://`; from iOS 6.14, PMTiles metadata is always interpreted as XYZ.
Android PMTiles sources participate in the ambient cache from 13.3.

### MLT vector tiles

MLT parsing starts in Android 12.1 and iOS 6.20. Android 13.4 additionally
supports FastPFOR-encoded MLT tiles.

### Synchronous source updates

Use synchronous GeoJSON updates only when the source change must be visible
before execution continues. They arrive in Android 12.3 and iOS 6.22, with the
Android 13 options-based migration shown above.

### Feature state

Vector-tile feature-state updates are complete in iOS 6.15 across
`GeometryTile` and `SourceFeatureState`; do not retain workarounds for the old
partial-update behavior. Android public feature-state functionality arrives in
13.4.

## High-value camera and rendering changes

Camera roll is available on iOS from 6.23 and Android from 13.0. Frustum offset
is available on iOS from 6.21 and Android from 12.2, allowing screen edges to be
omitted. iOS also supports maximum camera bounds and a configurable
north-snap threshold.

Color-relief layers and revised hillshade algorithms arrive in iOS 6.24 and
Android 13.0. Android 13.0.1 fixes those layers becoming invisible above fill
layers on Vulkan.

Android 13.1 changes pitched-map icon offsets by disabling icon scaling when
offsets are present. Visually recheck affected styles. Android 13.4 adds rounded
fill extrusions.

## Style compatibility guardrails

A valid version 8 style that works in GL JS is not automatically
feature-equivalent on Native. Check the separate Android and iOS support-table
entry, minimum version, and unsupported issue for each root property, source
option, layer property, and expression.

Native Android and iOS do not support root `centerAltitude`, root `roll`, or
root `state` and global-state functionality. Native pitch is limited to 0–60
degrees.

A `glyphs` URL works, but omitting it for local fonts does not work on Android
or iOS. Root `font-faces` has basic support from Android 11.13.0 and iOS 6.18.0.

Because `get` produces a generic value, assert the required type:

```json
["string", ["get", "feature_property"]]
```

Use a `to-*` coercion when conversion and fallback behavior are wanted:

```json
["to-number", ["get", "feature_property"], 0]
```

Wait for the style to load before mutating it through Android `Style` or iOS
`MLNStyle`. Use typed native property names rather than mechanically converting
JSON names.

## Renderer mental model

A platform Map View owns viewport and map configuration but does not render by
itself. Platform wrappers compile with the C++ core and a build-selected
backend. Tile workers parse and lay out source data; the render orchestrator
builds an ordered render tree; backend resource builders turn prepared buckets
into drawables.

Core work crosses threads as immutable actor messages. A worker pool prepares
tiles while one render loop draws available state. iOS renders on the UI thread;
Android uses a separate `GLSurfaceView` `GLThread` and batches UI changes for
that thread.

Drawables hold shared transition and tile-tracking state. Backend subclasses own
upload, binding, drawing, per-frame updates, and teardown. Design-level shader,
render-pass, and offscreen-target capabilities are not necessarily exposed by
every platform's public SDK.

## Backend selection

OpenGL ES 3 is stable on Android, Linux, Windows, Linux and Windows Node, and Qt
3. Stable Qt 3 supports only OpenGL. Vulkan is stable on Android and Linux, with
a macOS CMake route through MoltenVK. Metal is the stable recommended iOS
default and powers macOS Node rendering. WebGPU remains experimental.

Android source builds expose `opengl` and `vulkan` Gradle flavors, setting
`MLN_WITH_OPENGL=ON` and `MLN_WITH_VULKAN=ON`. iOS selects Metal in its CMake
or Bazel configuration.

## Platform workflow

1. Identify the platform package and exact release line.
2. Read the matching platform reference before changing dependencies or public
   API calls.
3. For style work, also read
   [references/styles-and-interoperability.md](references/styles-and-interoperability.md).
4. For backend, custom-layer, rendering, or threading work, read
   [references/architecture-and-rendering.md](references/architecture-and-rendering.md).
5. For Linux source builds or fixture failures, read
   [references/linux-build-and-testing.md](references/linux-build-and-testing.md).
6. Verify visual behavior with platform rendering tests when changing backend,
   pitch, offsets, glyphs, hillshade, or color-relief behavior.
