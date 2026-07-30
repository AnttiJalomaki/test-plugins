# Architecture and rendering

## Repository and release boundaries

The C++ core lives under `include/mbgl` and `src/mbgl`
(`core-architecture`). Android reaches it through JNI, iOS through the shared
Objective-C++ Darwin layer, and Node distributes it as
`@maplibre/maplibre-gl-native`. Part of the Qt binding lives in the separate
`maplibre-native-qt` repository.

Android, iOS, Node, and Qt have separate release streams. There is no single
public C++ core version that identifies all platform releases.

Android and iOS use semantic versioning but have no fixed release cadence or
LTS releases. To backport a fix, request an older-series branch, then submit
the fix and changelog change to a branch named `platform-x.x.x`, such as
`android-10.x.x`. After it merges, a release is attempted. Changes to the
release workflow may also need backporting.

## Build-time platform composition

CMake covers platform builds; Bazel is also used for iOS and several core
desktop targets (`core-architecture`). The build selects the renderer
backend. The platform wrapper and core renderer are compiled together rather
than combined as independently interchangeable runtime components.

## Map views and observers

A platform Map View owns viewport and map configuration but has no rendering
capability itself (`core-architecture`). Map observers handle configuration
and lifecycle changes including style, camera, idle, and render
start/completion. Rendering observers handle frame-level events, and
rendering events can propagate to map observers.

On Android, the Java Map View initializes the device renderer and a JNI-backed
native Map View peer (`render-pipeline`). The peer wraps the generic Map
component. Its native `MapRenderer` actor passes platform rendering events to
the core renderer.

`Transform` stores the combined global camera and viewport state; it does not
represent one operation. Observer notifications allow the renderer to derive
rotation, pitch, projection, resize, and camera transforms.

## Actor and render threading

Core rendering work crosses threads as immutable actor messages, including
callable messages delivered through typed mailboxes (`core-architecture`).
A worker pool prepares tiles while one render loop draws the currently
available state.

iOS runs the render loop on the UI thread. Android uses a separate
`GLSurfaceView` `GLThread` and batches UI changes for it. Each platform
provides the core concurrency primitives.

## Tile-worker contracts

Geometry, raster, and elevation workers do not inherit from a shared worker
base (`core-architecture`). Their corresponding tile types inherit from the
common `Tile` base. A worker is an actor that accepts its matching tile type.
Messages may run on any thread, provided only one thread processes a given
worker instance at a time.

## Glyph atlas

Glyphs are 24-pixel signed-distance-field bitmaps in a texture atlas packed in
a protobuf container (`core-architecture`). Pixels inside an outline have
values `192`–`255`; outside pixels have values `0`–`191`. This shared atlas
allows GPU resizing, rotation, and halo rendering.

## Tile cover and render-tree construction

For a tile source, `RenderSource::update` produces the tile pyramid selected
by the viewport's tile cover (`render-pipeline`). The render orchestrator
builds an ordered render tree from render layers, render sources, and
atlas-backed items but does not draw the tree itself.

Unchanged tiles remain reusable. Dirty tile or style state is updated.

## Geometry-tile coalescing

Work is queued per unique geometry tile (`render-pipeline`). Updates arriving
during parsing or layout are folded into the latest combined state for the
next pass instead of replaying every intermediate camera state.

Parsing discovers needed glyphs and images. When dependencies arrive, the
worker may enter `NeedsSymbolLayout` or `NeedsParse`. Finalization waits for
parsing and symbol dependencies before emitting geometry, resource
references, and collision metadata.

## Prepared tiles and backend resources

Preparation loads style resources, TileJSON, tiles, glyphs, and sprites
through the file source and cache (`render-pipeline`). Workers parse and lay
out source data layer by layer.

Prepared buckets become drawables and are uploaded through backend-specific
resource builders. Descriptions that end the pipeline at OpenGL buffers
predate this abstraction; OpenGL ES, Metal, and Vulkan consume the same
higher-level tile state.

## Drawable and Builder boundary

Layers provide shader selection, attribute arrays, uniforms or uniform
structs, and geometry to a backend-specific Builder (`render-pipeline`). The
Builder emits Drawables.

Shared Drawable state handles cross-backend concerns such as transitions and
tile tracking. Backend subclasses own upload and binding, direct or indirect
drawing, per-frame updates, and resource teardown.

## Shader modularization contract

The accepted renderer design uses a generic shader representation and a
thread-safe registry keyed by well-known names (`render-pipeline`). The
contract supports:

- shader source or references to precompiled shaders
- named uniforms or uniform structs
- calculation shaders
- adding or replacing a shader before a layer requests it

These are architectural requirements, not a promise that every operation is
available through every platform's public API.

## Render passes and offscreen targets

The modularization design calls for named, ordered passes whose outputs can
feed later passes; empty passes are omitted (`render-pipeline`).

Offscreen targets carry size and bit-depth settings, allow geometry to choose
targets, and support querying or snapshotting. Snapshots use callbacks after
drawing completes so non-OpenGL backends do not have to stall render flow.

## Stable backend matrix

Backend support is platform-specific (`rendering-platforms`):

| Backend | Stable platforms and constraints |
| --- | --- |
| OpenGL ES 3 | Android, Linux, Windows, Linux/Windows Node, and Qt 3; stable Qt 3 supports only OpenGL |
| Vulkan | Android and Linux; macOS has a CMake route through MoltenVK |
| Metal | Stable default and recommended iOS backend; used by macOS Node since Node 6.0 |
| WebGPU | Experimental |

## Source-build backend selection

Android source builds provide `opengl` and `vulkan` Gradle flavors
(`rendering-platforms`). They set `MLN_WITH_OPENGL=ON` and
`MLN_WITH_VULKAN=ON`, respectively. A repository checkout defaults to the
broad-compatibility OpenGL flavor. iOS selects Metal through CMake or Bazel
configuration.

## Linux OpenGL development build

On Ubuntu 22.04 or later, clone submodules and use the `linux-opengl` preset
(`rendering-platforms`). It builds GLFW development tools and can produce
static libraries for other C++ projects.

The preset defaults to Wayland and therefore also needs
`libegl1-mesa-dev`. `libsqlite3-dev` is optional because SQLite may be
vendored.

```bash
git clone --recurse-submodules -j8 https://github.com/maplibre/maplibre-native.git
cd maplibre-native
apt install build-essential clang cmake ccache ninja-build pkg-config
apt install libcurl4-openssl-dev libglfw3-dev libuv1-dev libpng-dev libicu-dev libjpeg-turbo8-dev libwebp-dev xvfb libegl1-mesa-dev
cmake --preset linux-opengl
cmake --build build-linux-opengl --target mbgl-render
```

## Linux image rendering

`mbgl-render` accepts a style URL or file and writes a PNG
(`rendering-platforms`). A local style may address an MBTiles database with
an absolute `mbtiles:///path/to/data.mbtiles` source URL.

```bash
./build-linux-opengl/bin/mbgl-render --style style.json --output out.png
```

On a remote or containerized host without an X display, install `xvfb` and
`xauth`, then run the renderer through a virtual display:

```bash
xvfb-run -a ./build-linux-opengl/bin/mbgl-render --style style.json --output out.png
```

## Linux render fixtures

Linux render tests compare each fixture's rendered PNG with `expected.png`
and leave `actual.png` and `diff.png` beside it (`rendering-platforms`). They
also generate an HTML summary next to the manifest. Run the whole manifest or
select fixtures with `--filter`:

```bash
./build-linux-opengl/mbgl-render-test-runner --manifestPath metrics/linux-clang8-release-style.json
./build-linux-opengl/mbgl-render-test-runner --manifestPath metrics/linux-clang8-release-style.json --filter "render-tests/fill-visibility/visible"
```
