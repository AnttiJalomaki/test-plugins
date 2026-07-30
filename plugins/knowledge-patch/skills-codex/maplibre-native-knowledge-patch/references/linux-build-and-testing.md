# Linux Build and Rendering Tests

## OpenGL development build

Use Ubuntu 22.04 or later, clone submodules, and select the `linux-opengl`
preset. It builds GLFW-based development tools and can produce static libraries
for other C++ projects.

The preset defaults to Wayland and therefore requires `libegl1-mesa-dev`.
`libsqlite3-dev` is optional because SQLite can be vendored.

```bash
git clone --recurse-submodules -j8 https://github.com/maplibre/maplibre-native.git
cd maplibre-native
apt install build-essential clang cmake ccache ninja-build pkg-config
apt install libcurl4-openssl-dev libglfw3-dev libuv1-dev libpng-dev libicu-dev libjpeg-turbo8-dev libwebp-dev xvfb libegl1-mesa-dev
cmake --preset linux-opengl
cmake --build build-linux-opengl --target mbgl-render
```

## Render a style to PNG

`mbgl-render` accepts a style URL or local file and writes a PNG:

```bash
./build-linux-opengl/bin/mbgl-render --style style.json --output out.png
```

A local style can reference an MBTiles database with an absolute
`mbtiles:///path/to/data.mbtiles` source URL.

## Render without an X display

On a remote or containerized host without an X display, install both `xvfb` and
`xauth`, then use a virtual display:

```bash
xvfb-run -a ./build-linux-opengl/bin/mbgl-render --style style.json --output out.png
```

## Run render fixtures

Linux render tests compare each fixture's output with `expected.png`. Failures
leave `actual.png` and `diff.png` beside the fixture and generate an HTML
summary next to the manifest.

Run the complete manifest:

```bash
./build-linux-opengl/mbgl-render-test-runner --manifestPath metrics/linux-clang8-release-style.json
```

Narrow the run with `--filter`:

```bash
./build-linux-opengl/mbgl-render-test-runner --manifestPath metrics/linux-clang8-release-style.json --filter "render-tests/fill-visibility/visible"
```
