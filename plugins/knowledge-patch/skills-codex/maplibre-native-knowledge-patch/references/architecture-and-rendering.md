# Architecture and Rendering

## Repository and release boundaries

The C++ core lives under `include/mbgl` and `src/mbgl`. Android reaches it
through JNI. iOS uses the shared Objective-C++ Darwin layer. Node packages it as
`@maplibre/maplibre-gl-native`. Part of the Qt binding is maintained in the
separate `maplibre-native-qt` repository.

Android, iOS, Node, and Qt have separate release streams. There is no single
public C++ core version that identifies every platform release.

Android and iOS use semantic versioning but have no fixed release cadence or
LTS releases. To backport a fix, request an older-series branch, submit the fix
and changelog update to `platform-x.x.x` such as `android-10.x.x`, and account
for any release-workflow changes that also require backporting. A release is
attempted after the backport merges.

## Build-time composition

CMake covers platform builds. Bazel is also used for iOS and several core
desktop targets. The renderer backend is selected at build time; the platform
wrapper and core renderer compile together rather than acting as interchangeable
runtime components.

## Map views and observers

A platform Map View owns viewport and map configuration but has no rendering
capability itself.

Map observers cover configuration and lifecycle changes such as style, camera,
idle, and render start or completion. Rendering observers cover frame-level
events, and rendering events can propagate to map observers.

## Actors and render threads

Core rendering work crosses threads as immutable actor messages, including
callable messages delivered through typed mailboxes. Each platform supplies the
core concurrency primitives.

A worker pool prepares tiles while one render loop draws the available state.
iOS runs the render loop on the UI thread. Android uses a separate
`GLSurfaceView` `GLThread` and batches UI changes for it.

## Tile-worker contracts

Geometry, raster, and elevation workers do not share a worker base class. Their
matching tile types inherit the common `Tile` base.

A worker is an actor accepting its matching tile type. Its messages may run on
any thread, provided only one thread processes a given worker instance at a
time.

## Android-to-core render handoff

Android's Java Map View initializes the device renderer and a JNI-backed native
Map View peer. That peer wraps the generic Map component. The native
`MapRenderer` actor passes platform rendering events to the core renderer.

`Transform` stores combined global camera and viewport state; it does not
represent a single operation. Observer notifications allow the renderer to
derive rotation, pitch, projection, resize, and camera transforms.

## Tile cover and render tree

For a tile source, `RenderSource::update` creates the tile pyramid selected by
the viewport tile cover.

The render orchestrator builds an ordered render tree from render layers,
render sources, and atlas-backed items, but does not draw it. Unchanged tiles
remain reusable while dirty tile or style state is updated.

## Geometry-tile coalescing

Work is queued per unique geometry tile. Updates arriving during parsing or
layout are folded into the newest combined state for the next pass instead of
replaying every intermediate camera state.

Parsing discovers required glyphs and images. Dependency arrivals may move the
worker through `NeedsSymbolLayout` or `NeedsParse`. Finalization waits for
parsing and symbol dependencies before emitting geometry, resource references,
and collision metadata.

## From source data to backend resources

The preparation path loads style resources, TileJSON, tiles, glyphs, and
sprites through the file source and cache. Workers parse and lay out source
data layer by layer.

Prepared buckets become drawables and are uploaded through backend-specific
resource builders. Descriptions that end the pipeline in OpenGL buffers predate
this abstraction; OpenGL ES, Metal, and Vulkan consume the same higher-level
tile state.

## Drawable and Builder boundary

Layers supply shader selection, attribute arrays, uniforms or uniform structs,
and geometry to a backend-specific Builder. The Builder emits Drawables.

Shared Drawable state handles cross-backend concerns such as transitions and
tile tracking. Backend subclasses own upload and binding, direct or indirect
drawing, per-frame updates, and resource teardown.

## Shader modularization contract

The accepted renderer design replaces opaque per-program handling with a
generic shader representation and a thread-safe registry keyed by well-known
names.

Its architectural contract supports:

- shader source or precompiled references;
- named uniforms or uniform structs;
- calculation shaders;
- adding or replacing a shader before a layer requests it.

These are design requirements, not a promise that each operation is exposed by
every platform's public API.

## Render passes and offscreen targets

The modularization design calls for named, ordered passes whose outputs may feed
later passes; empty passes are omitted.

Offscreen targets carry size and bit-depth settings, allow geometry to select
targets, and support querying or snapshotting. Snapshot callbacks run after
drawing completes, avoiding a requirement for non-OpenGL backends to stall the
render flow.

## Stable and experimental backends

OpenGL ES 3 is stable on Android, Linux, Windows, Linux and Windows Node builds,
and Qt 3. Stable Qt 3 supports only OpenGL.

Vulkan is stable on Android and Linux and has a macOS CMake route through
MoltenVK. Metal is the stable default and recommended iOS backend and has
powered macOS Node rendering since Node 6.0. WebGPU backends remain
experimental.

## Backend selectors

Android source builds provide `opengl` and `vulkan` Gradle flavors. They set
`MLN_WITH_OPENGL=ON` and `MLN_WITH_VULKAN=ON`, respectively. The checkout's
broad-compatibility default is OpenGL.

iOS selects Metal through CMake or Bazel configuration.
