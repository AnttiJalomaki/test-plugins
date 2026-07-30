# Node and Qt Bindings

## Node runtime and renderer

At the `node-6.1.0` boundary, Node.js 16 support is removed. The package moves
back to `@mapbox/node-pre-gyp`, which requires Node.js 18 or newer. Stable
binding 6.4.1 supports Node.js 20, 22, and 24; Node.js 26 support belongs to the
6.5 prerelease line.

Linux and Windows Node builds use the drawable renderer, and the legacy
renderer has been removed. Metal has powered macOS Node rendering since Node
6.0.

Node 6.1 supports PMTiles-backed map data. Sprites can define `textFitWidth` and
`textFitHeight`, exposing text-fit metadata during Node rendering.

## Node map rendering

`Map.render` accepts optional `zoom`, `width`, `height`, `[longitude, latitude]`
`center`, counter-clockwise-from-north `bearing`, `pitch`, and style `classes`.
It asynchronously returns a four-channel raw pixel buffer.

Defaults are zoom 0, 512×512 pixels, center `[0, 0]`, and zero bearing and
pitch. Calling `release()` permanently disables later renders. It is safe to
call inside the render callback because the returned buffer remains retained
for that callback.

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
style resource through application code. The handler must understand every
custom URL scheme in the style.

`kind` values are:

| Value | Kind |
| ---: | --- |
| 0 | `Unknown` |
| 1 | `Style` |
| 2 | `Source` |
| 3 | `Tile` |
| 4 | `Glyphs` |
| 5 | `SpriteImage` |
| 6 | `SpriteJSON` |

A successful response requires uncompressed byte `data` and may include
`modified` and `expires` dates plus an `etag`. Call the callback with no
arguments for a no-content result. Constructor `ratio` controls high-density
rendering scale.

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

## Node log events

The imported module is an `EventEmitter`. Its `message` events can include
`class`, `severity`, `code`, and `text`, making native style and resource
failures observable.

```js
const mbgl = require('@maplibre/maplibre-gl-native');

mbgl.on('message', (message) => {
  console.error(message.class, message.severity, message.code, message.text);
});
```

## Qt 3 libraries and CMake

Qt 3 places its API in `QMapLibre` and splits installation into `QMapLibre`,
`QMapLibreLocation`, and `QMapLibreWidgets`. The CMake package exposes Core,
Location, and Widgets components and `QMapLibre::*` targets.

Qt 3 supports static builds and CMake subproject consumption. Releases were
built with Qt 6.5–6.7 on all supported platforms and Qt 5.15.2 on macOS, Linux,
and Windows.

Point `QMapLibre_DIR` at `<install>/lib/cmake/QMapLibre`, or add the installation
prefix to `CMAKE_PREFIX_PATH`. Widgets deployments need both `QMapLibre` and
`QMapLibreWidgets`.

```cmake
find_package(QMapLibre COMPONENTS Widgets REQUIRED)
target_link_libraries(MyApplication PRIVATE QMapLibre::Widgets)
```

## Qt 3 QML and deployment

QML applications use `import MapLibre 3.0` and configure styles through
`maplibre.map.styles`. Deploy both `plugins/geoservices` and `qml/MapLibre`,
together with the core and Location libraries.

Linking Location exposes `qmaplibre_location_setup_plugins`, which installs both
plugin trees:

```cmake
find_package(QMapLibre COMPONENTS Location REQUIRED)
target_link_libraries(MyApplication PRIVATE QMapLibre::Location)
qmaplibre_location_setup_plugins(MyApplication)
```

For undeployed development runs, set `QML_IMPORT_PATH=<install>/qml` and
`QT_PLUGIN_PATH=<install>/plugins` if Qt cannot find the imports or plugins. Set
`QSG_RHI_BACKEND=opengl` to force the Qt 3 renderer. Stable Qt 3 supports only
OpenGL.

## Qt Android multi-ABI builds

Select the ABI-specific CMake package beneath a common `QMapLibre_Android_DIR`;
do not reuse one architecture's `QMapLibre_DIR`.

```cmake
if(ANDROID AND DEFINED ENV{QMapLibre_Android_DIR})
    set(QMapLibre_DIR
        "$ENV{QMapLibre_Android_DIR}/${ANDROID_ABI}/lib/cmake/QMapLibre")
endif()
```

## Empty Qt styles

Qt 3 permits a `Style` with an empty URL. A placeholder URL is no longer needed
just to construct the style object.
