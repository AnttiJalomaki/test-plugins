# Upgrades and runtime

Use this reference to choose an Electron target, inspect the embedded runtime,
and find migration work that crosses subsystems.

## Runtime stack and support milestones

| Electron | Chromium | Node.js | V8 | Support transition |
| --- | --- | --- | --- | --- |
| 34.0.0 | `132.0.6834.83` | `20.18.1` | `13.2` | Electron 31 reached end of support; supported lines were 34, 33, and 32 |
| 35.0.0 | `134.0.6998.44` | `22.14.0` | `13.5` | Electron 32 reached end of support; supported lines were 35, 34, and 33 |
| 36.0.0 | `136.0.7103.48` | `22.14.0` | `13.6` | Electron 33 reached end of support; supported lines were 36, 35, and 34 |
| 37.0.0 | `138.0.7204.35` | `22.16.0` | `13.8` | Electron 34 reached end of support; supported lines were 37, 36, and 35 |
| 38.0.0 | `140.0.7339.41` | `22.18.0` | `14.0` | Electron 35 reached end of support; supported lines were 38, 37, and 36 |
| 39.0.0 | `142.0.7444.52` | `22.20.0` | `14.2` | Electron 36 reached end of support; supported lines were 39, 38, and 37 |
| 40.0.0 | `144.0.7559.60` | `24.11.1` | `14.4` | Electron 37 reached end of support; supported lines were 40, 39, and 38 |
| 41.0.0 | `146.0.7680.65` | `24.14.0` | `14.6` | Electron 38 reached end of support; install 41.0.2 rather than the initial package |
| 42.0.0 | `148.0.7778.96` | `24.15.0` | `14.8` | Electron 39 reached end of support |
| 43.0.0 | `150.0.7871.46` | `24.17.0` | `15.0` | Electron 40 reached end of support |

Electron 34 moved from Electron 33's Chromium 130, Node.js `20.18.0`, and V8
13.0 to Chromium 132, Node.js `20.18.1`, and V8 13.2. The initial Electron
41.0.0 package was followed by high-priority fixes, which is why 41.0.2 is the
upgrade target.

The notable major jumps are Node.js 20 to 22 in Electron 35, Node.js 22 to 24
in Electron 40, and V8 13 to 14 in Electron 38. Test native modules, Node.js
flags, and JavaScript engine assumptions rather than treating an Electron-only
version change as isolated.

## Platform floors and binary availability

- Electron 38 requires macOS 12 Monterey or later; Electron 37 and older can
  still run on macOS 11 Big Sur.
- Electron 44 requires macOS 13 Ventura or later; older Electron lines can
  continue to run on macOS 12.
- Electron 43 is the final series with prebuilt Windows x86 (`win32-ia32`) and
  Linux ARMv7 (`linux-armv7l`) binaries. Support ends after that series reaches
  end of life in January 2027.
- Electron 44 stops publishing 32-bit `chromedriver`, `mksnapshot`, and
  `ffmpeg`, and stops publishing Windows x86 `node.lib` on the headers CDN.

## Package installation and update formats

Since 42.0.0, the `electron` npm package downloads its binary the first time
its main bin script runs rather than from `postinstall`. An install may disable
scripts, then fetch explicitly:

```sh
npm install electron --save-dev --ignore-scripts
npx install-electron
```

`ELECTRON_SKIP_BINARY_DOWNLOAD` is removed. Use `ELECTRON_INSTALL_PLATFORM`
and `ELECTRON_INSTALL_ARCH` to fetch for another target.

`autoUpdater` supports MSIX packages. An update service can publish MSIX and
Squirrel.Mac updates with essentially the same JSON format. This capability
landed in 41.0.0 and was backported to Electron 39.5.0 and 40.2.0.

On macOS, debug-symbol consumers must handle `dsym.tar.xz` rather than
`dsym.zip` since 40.0.0.

## Operating-system and command-line transitions

### Linux display and toolkit behavior

Electron 36.0.0 defaults to GTK 4 on GNOME. A process that also loads GTK 2/3
symbols can fail because the versions cannot coexist. Select GTK 3 before app
readiness when required:

```js
app.commandLine.appendSwitch('gtk-version', '3');
```

Electron 38.0.0 removes `ELECTRON_OZONE_PLATFORM_HINT`; Chromium's
`--ozone-platform` defaults to `auto`. A Wayland session therefore launches a
native Wayland application. Pass `--ozone-platform=x11` for the former
Xwayland behavior.

Electron 38.0.0 no longer replaces `XDG_CURRENT_DESKTOP` with `Unity`; it
contains the real desktop environment. `ORIGINAL_XDG_CURRENT_DESKTOP` is
removed.

### Command-line ownership

Since 36.0.0, `app.commandLine` lowercases uppercase switches and arguments.
It is for case-insensitive Chromium switches. Read application-specific,
case-sensitive arguments from `process.argv`.

Chromium is deprecating `--host-rules`; use `--host-resolver-rules` from
39.0.0 onward.

## Cross-cutting removals and deprecations

- Electron 35.0.0 deprecates `systemPreferences.isAeroGlassEnabled()` with no
  replacement. Electron 36 removes it. It had returned `true` since Electron
  23 because supported Windows versions do not allow DWM composition to be
  disabled; remove the conditional code.
- Electron 36.0.0 deprecates `NativeImage.getBitmap()`; call `toBitmap()`.
- Electron 38.0.0 removes the `webContents` `plugin-crashed` event.
- Electron 40.0.0 deprecates direct renderer access to `clipboard`. Move calls
  to a preload and expose a minimal bridge. Electron 44 removes the renderer
  Electron module; use `navigator.clipboard` for ordinary operations or the
  bridge for advanced operations.
- Electron 41.0.0 deprecates the Linux `showHiddenFiles` dialog option.
  Electron 43.0.0 no longer supports it on Linux; macOS and Windows support
  remain.
- Electron 42.0.0 deprecates an `hslShift` array passed directly as the second
  argument to `nativeImage.createFromNamedImage()`. Pass an options object:

```js
nativeImage.createFromNamedImage(imageName, {
  hslShift: [0, 1, -1],
});
```

Subsystem-specific migrations for session preloads, service workers,
extensions, frame tokens, storage quotas, console events, printer properties,
and protocol response sessions are documented in the other references.
