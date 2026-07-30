# Graphics, media, and native image

## Offscreen rendering and shared textures

Electron 34.0.0 adds GPU-accelerated shared-texture offscreen rendering.

HDR offscreen rendering supports `RGBAF16` in the scRGB HDR color space since
39.0.0.

In 39.0.0, the shared-texture `paint` event payload changes shape:
`sharedTextureHandle`, `planes`, and `modifier` move under a structured
`handle` property instead of appearing at the top level.

Electron 40.0.0 can import an external shared texture as a `VideoFrame`.
Imported shared textures add NV12 support in 41.0.0, also available in
Electron 40. Electron 42.0.0 adds `nv16` and the 10-bit YUV format `p010le`.

Offscreen rendering defaults to a constant device scale factor of `1.0` since
42.0.0 instead of inheriting the primary display's scale. Set the required
scale explicitly:

```js
const window = new BrowserWindow({
  webPreferences: {
    offscreen: { deviceScaleFactor: 2 },
  },
});
```

## Desktop capture audio

On macOS 14.2 and later, an app using `desktopCapturer` needs
`NSAudioCaptureUsageDescription` in `Info.plist` since Electron 39.0.0.
Without it, the newer CoreAudio Tap path produces a silent stream without an
error or warning. As a temporary compatibility measure, select the old path
before readiness:

```js
app.commandLine.appendSwitch(
  'disable-features',
  'MacCatapLoopbackAudioForScreenShare',
);
```

## NativeImage API migrations

`NativeImage.getBitmap()` is deprecated since 36.0.0. `toBitmap()` is the
equivalent preferred API; both return a newly allocated bitmap copy.

`nativeImage.createFromNamedImage()` accepts SF Symbol names since 40.0.0;
the support also exists in Electron 39.

Electron 42.0.0 deprecates passing an `hslShift` array directly as the second
argument to `createFromNamedImage()`. Use an options object:

```js
nativeImage.createFromNamedImage(imageName, {
  hslShift: [0, 1, -1],
});
```

## Color normalization and bitmap output

Since 43.0.0, input images with color profiles are normalized to sRGB inside
`nativeImage`. Pixel values from visually identical images should therefore
be similar after their profiles are applied.

In Electron 43, `toBitmap()` and deprecated `getBitmap()` also normalize
output to sRGB by default. Pass a `colorSpace` to retain the source space or
request another conversion:

```js
const { nativeImage } = require('electron');
const image = nativeImage.createFromPath('photo.png');
const p3Bitmap = image.toBitmap({
  colorSpace: {
    primaries: 'p3',
    transfer: 'srgb',
    matrix: 'rgb',
    range: 'full',
  },
});
```

Byte-for-byte image assertions must state the requested color space. A visual
match does not imply identical pre-conversion source bytes.
