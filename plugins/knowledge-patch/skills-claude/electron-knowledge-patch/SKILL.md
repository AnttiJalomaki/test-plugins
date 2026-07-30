---
name: electron-knowledge-patch
description: Electron
version: 43.0.0
license: MIT
metadata:
  author: Nevaberry
---


# Electron Knowledge Patch

Use this skill when upgrading, packaging, or implementing an Electron
application whose behavior depends on recent Electron APIs and embedded
Chromium, Node.js, or V8 changes. Check the application's pinned Electron
version before applying version-specific advice. Prefer the application's
manifest, lockfile, code, and tests when they disagree with general guidance.

## Reference index

| Reference | Topics |
| --- | --- |
| [upgrades-and-runtime.md](references/upgrades-and-runtime.md) | Runtime stacks, support lifecycle, operating-system floors, binary installation, companion artifacts |
| [sessions-networking-and-extensions.md](references/sessions-networking-and-extensions.md) | Sessions, preloads, service workers, web requests, protocols, storage, extensions, WebUSB and Web Serial |
| [processes-diagnostics-and-runtime.md](references/processes-diagnostics-and-runtime.md) | Frame lifecycle, utility processes, Node.js flags, crash and OOM diagnostics, cookies, PDF frames |
| [windows-input-and-platform-ui.md](references/windows-input-and-platform-ui.md) | BrowserWindow, menus, navigation, dialogs, shortcuts, Wayland/X11, appearance and platform UI |
| [graphics-media-and-native-image.md](references/graphics-media-and-native-image.md) | Offscreen rendering, shared textures, color spaces, NativeImage, CSS corners and desktop capture |
| [notifications-security-and-packaging.md](references/notifications-security-and-packaging.md) | Notifications, ASAR integrity, WebAuthn, safe storage, clipboard isolation, MSIX and signing |

## Breaking changes to handle first

### Prepare for the next major release

- Electron 44 requires macOS 13 or later; do not ship it to macOS 12.
- Electron 44 removes the Electron `clipboard` module from renderer
  processes. Use `navigator.clipboard` for ordinary access, or expose narrowly
  scoped preload functions through `contextBridge`.
- Electron 44 stops publishing 32-bit `chromedriver`, `mksnapshot`, and
  `ffmpeg`, and stops publishing Windows x86 `node.lib` on the headers CDN.
- Electron 43 is the final series with Windows x86 and Linux ARMv7 prebuilt
  Electron binaries. Plan a supported architecture migration before its
  January 2027 end of life.

### Replace removed and deprecated APIs

- Replace `Session.setPreloads()` and `getPreloads()` with
  `registerPreloadScript()`, `unregisterPreloadScript()`, and
  `getPreloadScripts()`.
- Replace `session.serviceWorkers.fromVersionID()` with
  `getInfoFromVersionID()` or `getWorkerFromVersionID()`, according to whether
  information or a `ServiceWorkerMain` object is needed.
- Move extension operations and events from `Session` to
  `session.extensions`.
- Replace `NativeImage.getBitmap()` with `toBitmap()`.
- Replace `webFrame.routingId` and `findFrameByRoutingId()` with
  `frameToken`, `findFrameByToken()`, and main-process
  `webFrameMain.fromFrameToken()`.
- Remove `quota`/`quotas` options from `Session.clearStorageData()` and remove
  the obsolete `syncable` quota type.
- Replace Chromium's `--host-rules` with `--host-resolver-rules`.
- Remove `webContents` `plugin-crashed` handlers and
  `systemPreferences.isAeroGlassEnabled()` branches.
- On Linux, stop passing `showHiddenFiles` to file dialogs.

### Account for changed defaults

- On Wayland, Electron chooses native Wayland automatically. Pass
  `--ozone-platform=x11` only when Xwayland behavior is required.
- GTK 4 is the default on GNOME. Use `--gtk-version=3` before startup only
  when an application must coexist with GTK 2/3 symbols.
- Offscreen rendering uses a constant device scale factor of `1.0`; set
  `webPreferences.offscreen.deviceScaleFactor` explicitly for other output
  scales.
- Downloads and dialogs without `defaultPath` now start in Downloads, falling
  back to Home. Persist and pass a directory explicitly to preserve a
  last-used-directory workflow.
- Linux frameless windows have rounded corners by default, and their Window
  Controls Overlay follows the native title-bar layout. Use
  `roundedCorners: false` when required and size content with the
  `titlebar-area-*` CSS environment values.
- `window.open()` popups are always resizable unless
  `setWindowOpenHandler()` overrides `resizable`.

## High-value migration patterns

### Isolate clipboard access

Keep privileged clipboard operations in a preload:

```js
const { clipboard, contextBridge } = require('electron');

contextBridge.exposeInMainWorld('clipboardAPI', {
  readText: () => clipboard.readText(),
});
```

Expose only the operations the page needs. Prefer the browser Clipboard API
when it is sufficient.

### Register preloads independently

Use per-script registration so libraries do not overwrite one another's
preload lists. Select `frame` or `service-worker` as the registration type.
Service-worker preloads use `ipcRenderer`; the main process receives their IPC
through `ServiceWorkerMain.ipc`.

### Preserve popup resize policy

Translate the feature string into explicit window options:

```js
webContents.setWindowOpenHandler((details) => ({
  action: 'allow',
  overrideBrowserWindowOptions: {
    resizable: details.features.includes('resizable=yes'),
  },
}));
```

### Make Linux title bars adaptive

Do not assume which side contains window controls:

```css
.titlebar-content {
  left: env(titlebar-area-x, 0px);
  width: env(titlebar-area-width, 100%);
}
```

### Diagnose a hung renderer

Enable `DocumentPolicyIncludeJSCallStacksInCrashReports` before the app is
ready and serve the renderer with
`Document-Policy: include-js-call-stacks-in-crash-reports`. An
`unresponsive` handler can then await
`webContents.mainFrame.collectJavaScriptCallStack()`.

### Handle utility-process behavior deliberately

- Listen for the utility process `error` event to capture fatal V8 diagnostic
  reports.
- An unhandled rejection now warns instead of crashing. Install a handler
  that calls `process.exit(1)` when fail-fast behavior is required.
- `process.exit()` is synchronous, so pending output may not flush.
- Accept `"memory-eviction"` anywhere child-process exit reasons are decoded.

## Packaging and security checkpoints

- Stable ASAR integrity terminates an application when its packaged ASAR hash
  is absent or mismatched. On macOS, use `@electron/asar` 4.1.0 or later to
  embed an integrity digest, then re-sign the application.
- Code-sign macOS applications that display notifications; unsigned
  applications receive the Notification `failed` event.
- Add `NSAudioCaptureUsageDescription` for desktop-capture audio on macOS
  14.2 or later.
- Use `app.configureWebAuthn({ touchID: { keychainAccessGroup } })` for the
  macOS Touch ID platform authenticator.
- The `electron` npm package downloads its binary on first execution rather
  than in `postinstall`. With `--ignore-scripts`, run `npx install-electron`
  explicitly. Do not use the removed `ELECTRON_SKIP_BINARY_DOWNLOAD`.
- macOS debug-symbol tooling must consume `dsym.tar.xz`, not `dsym.zip`.

## Rendering and image checkpoints

- Shared-texture offscreen rendering is GPU accelerated. Its `paint` payload
  nests `sharedTextureHandle`, `planes`, and `modifier` under `handle`.
- Imported textures can become `VideoFrame` objects and support NV12, NV16,
  and P010LE formats; offscreen output can use RGBAF16 in scRGB HDR.
- Images with profiles are normalized to sRGB. `NativeImage.toBitmap()`
  defaults to sRGB conversion; pass a `colorSpace` when source-space output or
  another conversion is required.
- Pass `hslShift` inside an options object to
  `nativeImage.createFromNamedImage()`.

## Platform behavior checkpoints

- On Windows, fullscreen hides the menu bar. `query-session-end` and improved
  `session-end` events cover session shutdown.
- On macOS, parented message dialogs center on the parent, and notifications
  use the signed-app-only `UNNotification` path.
- Read application-specific command-line arguments from `process.argv`;
  `app.commandLine` lowercases switches and arguments.
- For file-dialog portals older than version 4, `defaultPath` is unsupported.
  Require portal version 4 when that option is essential.
- Treat `XDG_CURRENT_DESKTOP` as the real desktop environment; do not expect
  Electron to replace it with `Unity` or provide
  `ORIGINAL_XDG_CURRENT_DESKTOP`.

## Working method

1. Read the pinned Electron version and the target operating systems and
   architectures.
2. Start with breaking changes, removed APIs, platform floors, and changed
   defaults.
3. Open the topic reference for the subsystem being changed.
4. Apply only behavior available in the pinned version, including backported
   availability when noted.
5. Exercise platform-specific paths on their real backend: native Wayland
   versus Xwayland, portal versus native dialogs, and signed versus unsigned
   macOS bundles.
6. Test packaging, preload isolation, process failures, and renderer lifecycle
   behavior in addition to the happy path.
