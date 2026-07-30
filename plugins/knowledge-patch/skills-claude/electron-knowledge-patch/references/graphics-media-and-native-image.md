# Graphics, Media, and NativeImage

## Offscreen rendering and shared textures

- Electron 34 adds GPU-accelerated shared-texture offscreen rendering.
- Since 39.0.0, offscreen rendering supports `RGBAF16` output in the scRGB
  HDR color space.
- Since 39.0.0, a shared-texture offscreen-rendering `paint` event supplies a
  structured payload: `sharedTextureHandle`, `planes`, and `modifier` are
  nested under `handle`, not exposed at the top level.
- Since 40.0.0, an external shared texture can be imported as a `VideoFrame`.
- The 41.0.0 notes record NV12 support for imported shared textures; it is
  also available in Electron 40.
- Since 42.0.0, imported shared textures additionally support the `nv16` and
  10-bit YUV `p010le` pixel formats.
- Since 42.0.0, offscreen rendering defaults to the constant device scale
  factor `1.0` instead of inheriting the primary display. Set the requested
  scale explicitly:

  ```js
  const window = new BrowserWindow({
    webPreferences: {
      offscreen: { deviceScaleFactor: 2 },
    },
  });
  ```

## NativeImage bitmap and color behavior

`NativeImage.getBitmap()` is deprecated since 36.0.0; use
`NativeImage.toBitmap()`. Both are aliases that allocate and return a bitmap
copy.

In Electron 43, images with color profiles passed to `nativeImage` have their
pixel values normalized to sRGB, making pixel values from visually identical
images comparable after applying their profiles. `toBitmap()` and deprecated
`getBitmap()` also convert output to sRGB by default. Pass `colorSpace` to
retain the source color space or choose a conversion:

```js
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

The 40.0.0 notes record SF Symbol names for
`nativeImage.createFromNamedImage()`; this support is also present in Electron
39.

Since 42.0.0, passing an `hslShift` array directly as the second argument to
`nativeImage.createFromNamedImage()` is deprecated. Put it in an options
object:

```js
nativeImage.createFromNamedImage(imageName, {
  hslShift: [0, 1, -1],
});
```

## CSS corners and smoothing

The custom `-electron-corner-smoothing` property, introduced in Electron 36
and documented with 37.0.0, turns a rounded corner into a continuous
squircle-like curve. It accepts `0%` through `100%`; `system-ui` resolves to
60% on macOS and 0% elsewhere:

```css
.box {
  border-radius: 24px;
  -electron-corner-smoothing: system-ui;
}
```

The smoothing also applies to borders, outlines, and shadows.

## Desktop-capture audio

Since 39.0.0, applications using `desktopCapturer` for desktop-capture audio
on macOS 14.2 and later require `NSAudioCaptureUsageDescription` in
`Info.plist`. Without it, the newer CoreAudio Tap path produces a dead audio
stream without an error or warning. As a temporary compatibility measure,
disable that path before startup:

```js
app.commandLine.appendSwitch(
  'disable-features',
  'MacCatapLoopbackAudioForScreenShare',
);
```
