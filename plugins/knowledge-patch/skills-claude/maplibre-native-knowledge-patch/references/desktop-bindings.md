# Desktop bindings

## Node.js upgrades

### PMTiles, sprites, and renderer

MapLibre Native Node 6.1 supports PMTiles-backed map data
(`node-6.1.0`). Sprites can expose `textFitWidth` and `textFitHeight`, making
their text-fit metadata available to Node rendering.

Linux and Windows builds now use the drawable renderer; the legacy renderer
has been removed.

### Runtime support

The package moved back to `@mapbox/node-pre-gyp`, which requires Node.js 18 or
newer. Node.js 16 is therefore unsupported (`node-6.1.0`).

Stable binding 6.4.1 supports Node.js 20, 22, and 24. Node.js 26 support
belongs to the 6.5 prerelease line and should not be assumed for the stable
package (`desktop-bindings`).

## Node rendering and lifecycle

`Map.render` accepts these optional fields (`desktop-bindings`):

- `zoom`, default `0`
- `width` and `height`, default `512` by `512`
- `center` as `[longitude, latitude]`, default `[0, 0]`
- `bearing`, counter-clockwise from north and default `0`
- `pitch`, default `0`
- style `classes`

Rendering is asynchronous and yields a raw four-channel pixel buffer.
`release()` permanently disables subsequent renders. It is safe to call
`release()` in the render callback because the buffer stays retained for that
callback.

```js
const mbgl = require('@maplibre/maplibre-gl-native');
const map = new mbgl.Map();

map.load(require('./style.json'));
map.render({ width: 256, height: 256, center: [24.94, 60.17], zoom: 10 },
  (error, rgba) => {
    if (error) throw error;
    map.release();
    // rgba contains raw four-channel pixel data.
  });
```

## Node resource requests

The `Map` constructor's `request({ url, kind }, callback)` hook routes every
style resource through application code (`desktop-bindings`). Resource kinds
are:

| Kind | Value |
| --- | ---: |
| `Unknown` | 0 |
| `Style` | 1 |
| `Source` | 2 |
| `Tile` | 3 |
| `Glyphs` | 4 |
| `SpriteImage` | 5 |
| `SpriteJSON` | 6 |

The handler must understand every custom URL scheme used by the style. A
successful result requires uncompressed byte `data` and may include
`modified` and `expires` dates plus an `etag`. Calling the callback with no
arguments represents no content. Constructor option `ratio` controls the
high-density rendering scale.

```js
const fs = require('node:fs');
const path = require('node:path');
const mbgl = require('@maplibre/maplibre-gl-native');

const map = new mbgl.Map({
  request({ url }, callback) {
    fs.readFile(path.join('base/path', url), (error, data) => {
      if (error) return callback(error);
      callback(null, { data });
    });
  },
  ratio: 2
});
```

## Node logging

The imported module is an `EventEmitter`. Its `message` events may contain
`class`, `severity`, `code`, and `text`, exposing native style and resource
failures (`desktop-bindings`):

```js
const mbgl = require('@maplibre/maplibre-gl-native');

mbgl.on('message', (message) => {
  console.error(message.class, message.severity, message.code, message.text);
});
```

## Qt 3 libraries and CMake targets

Qt 3 places the API in the `QMapLibre` namespace and splits installation into
`QMapLibre`, `QMapLibreLocation`, and `QMapLibreWidgets`
(`desktop-bindings`). The CMake package exposes them as the `Core`, `Location`,
and `Widgets` components and as `QMapLibre::*` targets.

Qt 3 supports static builds and use as a CMake subproject. Its release was
built with Qt 6.5–6.7 on all supported platforms, plus Qt 5.15.2 on macOS,
Linux, and Windows.

Point `QMapLibre_DIR` to `<install>/lib/cmake/QMapLibre`, or put the
installation prefix in `CMAKE_PREFIX_PATH`. A Widgets deployment must include
both the `QMapLibre` and `QMapLibreWidgets` libraries.

```cmake
find_package(QMapLibre COMPONENTS Widgets REQUIRED)
target_link_libraries(MyApplication PRIVATE QMapLibre::Widgets)
```

## Qt 3 QML and deployment

Qt 3 QML applications use `import MapLibre 3.0` and configure styles through
`maplibre.map.styles` (`desktop-bindings`).

Deploy both `plugins/geoservices` and `qml/MapLibre`, together with the core
and Location libraries. Linking the Location component makes
`qmaplibre_location_setup_plugins` available to install both plugin trees:

```cmake
find_package(QMapLibre COMPONENTS Location REQUIRED)
target_link_libraries(MyApplication PRIVATE QMapLibre::Location)
qmaplibre_location_setup_plugins(MyApplication)
```

For an undeployed development run, set `QML_IMPORT_PATH=<install>/qml` and
`QT_PLUGIN_PATH=<install>/plugins` when Qt cannot find them. Set
`QSG_RHI_BACKEND=opengl` to force the Qt 3 renderer.

## Qt Android multi-ABI selection

For a multi-ABI Android build, select the ABI-specific package below a common
`QMapLibre_Android_DIR`; do not reuse one architecture's `QMapLibre_DIR`
(`desktop-bindings`):

```cmake
if(ANDROID AND DEFINED ENV{QMapLibre_Android_DIR})
    set(QMapLibre_DIR
        "$ENV{QMapLibre_Android_DIR}/${ANDROID_ABI}/lib/cmake/QMapLibre")
endif()
```

## Empty Qt styles

Qt 3 permits constructing a `Style` with an empty URL. A placeholder URL is
no longer needed solely for construction (`desktop-bindings`).
