---
name: electron-knowledge-patch
description: Electron
version: 43.0.0
license: MIT
metadata:
  author: Nevaberry
---


# Electron Knowledge Patch

Use this skill when upgrading, reviewing, or debugging an Electron application
whose behavior may depend on recent Electron, Chromium, Node.js, V8, or
operating-system integration changes. Start with migrations, then open the
topic reference for the subsystem being changed.

Treat the application's manifest, lockfile, packaging configuration, code, and
tests as authoritative. Confirm the installed Electron maintenance release and
the target operating systems before applying version-sensitive advice.

## Reference index

| Reference | Topics |
| --- | --- |
| [Upgrades and runtime](references/upgrades-and-runtime.md) | Runtime stack, supported release lines, operating-system floors, binary installation and distribution, removed and deprecated APIs |
| [Sessions, protocols, and extensions](references/sessions-protocols-and-extensions.md) | Preloads, service workers, web requests, storage, permissions, protocols, WebUSB, Web Serial, WebSocket, extensions |
| [Windows, menus, and platform integration](references/windows-menus-and-platform-integration.md) | `BrowserWindow`, menus, dialogs, notifications, shortcuts, printing, navigation, Linux, macOS, and Windows behavior |
| [Processes, security, and diagnostics](references/processes-security-and-diagnostics.md) | Frames, utility processes, ASAR integrity, command-line flags, memory, tracing, OOM, safe storage, and fuses |
| [Graphics, media, and native image](references/graphics-media-and-native-image.md) | Offscreen rendering, shared textures, HDR, capture audio, pixel formats, `NativeImage`, and color spaces |

## Upgrade triage

1. Read `package.json` and the lockfile to find the exact Electron release.
2. Check packaging targets, minimum macOS version, Linux display backend, and
   any 32-bit artifacts before changing the dependency.
3. Search for removed and deprecated APIs listed below.
4. Audit behavior-sensitive defaults: download and dialog directories,
   Wayland selection, offscreen scale factor, popup resizing, and Linux window
   decorations.
5. Re-test preload boundaries, session registration, custom protocols,
   extensions, notification activation, printing, media capture, and native
   image pixel comparisons.
6. Test signed packaged applications on every target OS. Several integrations
   behave differently from an unpackaged development run.

## Breaking changes first

### Prepare for Electron 44

- Raise the deployment target to macOS 13 Ventura. Electron 44 does not run on
  macOS 12 Monterey.
- Stop expecting 32-bit `chromedriver`, `mksnapshot`, or `ffmpeg` companions,
  or Windows x86 `node.lib` from the headers CDN.
- Remove renderer imports of Electron's `clipboard` module. Use
  `navigator.clipboard` for ordinary web access or expose narrowly scoped
  preload operations through `contextBridge`.

### Account for Electron 43 behavior

- The default download destination is Downloads, with Home as fallback.
- Open and save dialogs without `defaultPath` also start in Downloads or Home
  instead of restoring the OS's last-used directory. Persist the selected
  directory and pass it explicitly when continuity matters.
- `nativeImage` normalizes profiled input to sRGB. `toBitmap()` also converts
  to sRGB by default; pass a `colorSpace` when source-space bytes are required.
- Linux frameless windows gain rounded corners by default. Disable them with
  `roundedCorners: false`.
- Linux Window Controls Overlay follows the native title-bar layout. Position
  content with `env(titlebar-area-x, 0px)` and
  `env(titlebar-area-width, 100%)`.
- Linux no longer supports the `showHiddenFiles` dialog option.
- Windows x86 and Linux ARMv7 prebuilt binaries end after the Electron 43
  release series reaches end of life.

### Check platform transitions

- Electron 38 requires macOS 12 or later and uses native Wayland by default in
  Wayland sessions. Use `--ozone-platform=x11` only when Xwayland is required.
- Electron 36 defaults to GTK 4 on GNOME. If incompatible GTK 2/3 symbols are
  loaded, select GTK 3 before readiness with the `gtk-version` switch.
- Electron 38 preserves the real `XDG_CURRENT_DESKTOP`; do not read the removed
  `ORIGINAL_XDG_CURRENT_DESKTOP` compatibility variable.
- `systemPreferences.isAeroGlassEnabled()` is gone. Delete feature branches
  based on it; the method had returned `true` on supported Windows versions.

## Deprecation and removal map

| Old interface or behavior | Replacement or action |
| --- | --- |
| `Session.setPreloads()` / `getPreloads()` | Register scripts individually with `registerPreloadScript()`, `unregisterPreloadScript()`, and `getPreloadScripts()` |
| `serviceWorkers.fromVersionID()` | Use `getInfoFromVersionID()` or `getWorkerFromVersionID()` |
| Positional `console-message` arguments | Read `level`, `message`, `lineNumber`, `sourceId`, and `frame` from the event object |
| Empty `WebRequestFilter.urls` for all requests | Use `urls: ['<all_urls>']` explicitly |
| `PrinterInfo.isDefault` / `status` | Removed; do not depend on these Chromium properties |
| `NativeImage.getBitmap()` | Use `toBitmap()` |
| Extension methods and events on `Session` | Use `session.extensions` |
| `clearStorageData({ quota: ... })` | Omit the deprecated singular `quota` option |
| `clearStorageData({ quotas: ... })` | Remove the later plural `quotas` object |
| App arguments through `app.commandLine` | Read case-sensitive app arguments from `process.argv` |
| `ProtocolResponse.session: null` | Pass an explicit session; use a unique partition only when isolation is required |
| `webFrame.routingId` / `findFrameByRoutingId()` | Use `frameToken`, `findFrameByToken()`, and `webFrameMain.fromFrameToken()` |
| `webContents` `plugin-crashed` event | Removed |
| Chromium `--host-rules` | Use `--host-resolver-rules` |
| Renderer `clipboard` | Use a preload bridge, then migrate to `navigator.clipboard` or bridged advanced operations for Electron 44 |
| Linux dialog `showHiddenFiles` | Remove it; deprecated in Electron 41 and unsupported in Electron 43 |
| Positional `hslShift` array | Pass `{ hslShift: [...] }` to `createFromNamedImage()` |

## High-value API patterns

### Register frame and service-worker preloads independently

Use `session.registerPreloadScript()` with `type: 'frame'` or
`type: 'service-worker'`. Service-worker preloads use `ipcRenderer`; the main
process communicates through `ServiceWorkerMain.ipc`. Use
`startWorkerForScope()` when a worker must be started deliberately and observe
`running-status-changed` for lifecycle updates.

This avoids replacing an entire session preload list when one library adds or
removes its own script.

### Diagnose an unresponsive renderer

Enable `DocumentPolicyIncludeJSCallStacksInCrashReports`, serve the document
with `Document-Policy: include-js-call-stacks-in-crash-reports`, and call
`webContents.mainFrame.collectJavaScriptCallStack()` from an `unresponsive`
handler. Check `WebFrameMain.detached` and `isDestroyed()` before using a frame.

### Intercept input before delivery

Use `WebContents` `before-mouse-event` to inspect and prevent mouse input.
Use `webPreferences.focusOnNavigation: false` when navigation must not
automatically focus the contents.

### Restore and size windows deliberately

- Restore history with `webContents.navigationHistory.restore(index, entries)`.
- `window.open()` recognizes `innerWidth` and `innerHeight`.
- Popups are always resizable by default. Use `setWindowOpenHandler()` and
  `overrideBrowserWindowOptions.resizable` when the feature string controls
  policy.

### Handle notifications as an application entry point

Code-sign macOS applications or notification delivery fails. On Windows, use
`Notification.handleActivation()` for clicks, replies, and action buttons that
may cold-start the application. Use notification IDs and groups for stable
history and removal behavior.

### Make request matching explicit

Use `'<all_urls>'` rather than an empty `urls` array. Add `excludeUrls` when a
broad filter has exceptions. Use `bypassCustomProtocolHandlers` on
`net.request` only for requests that must skip application protocol handlers.

### Secure custom protocols and extensions

When Chrome extensions must run on an application protocol, register the
scheme before readiness with `standard`, `secure`, and `allowExtensions`
privileges. Re-test CSS injection into accessible fallback frames such as
`about:blank` and `data:` and narrow targets where needed.

### Pin offscreen-rendering assumptions

Offscreen rendering defaults to a device scale factor of `1.0`. Set
`webPreferences.offscreen.deviceScaleFactor` explicitly for higher-density
output. Shared-texture consumers must read texture metadata from the `handle`
object and support only the formats their import path understands.

## Diagnostic routing

| Symptom | First checks |
| --- | --- |
| Renderer hangs without a useful stack | Document Policy header, feature switch, `collectJavaScriptCallStack()` |
| Utility process exits unexpectedly | `error`, `child-process-gone` reason including `memory-eviction`, explicit unhandled-rejection policy |
| Preload or extension stopped running | Per-script registration type, service-worker state, `session.extensions`, dynamic-import isolation requirements |
| Request hook sees nothing | Explicit `'<all_urls>'`, `excludeUrls`, session ownership, custom-protocol bypass |
| Window appearance changed on Linux | Wayland versus X11, GTK version, rounded corners, shadow, Window Controls Overlay environment values |
| Notification works in development but not packaged macOS | Code signing and notification failure event |
| Desktop capture has silent audio on macOS | `NSAudioCaptureUsageDescription` and CoreAudio capture path |
| Offscreen output size or colors changed | Explicit scale factor, shared-texture payload shape, pixel format, sRGB normalization |
| Dialog opens in an unexpected directory | Explicit `defaultPath`, Downloads/Home fallback, portal version |
| Native image byte comparisons fail | Input color profile and requested `toBitmap({ colorSpace })` |

## Working principles

- Apply a note only when the installed Electron version reaches the stated
  behavior; backported features are called out in the references.
- Distinguish Electron's main process, renderer, preload, service worker, and
  utility process. An API available in one context is not implicitly safe or
  supported in another.
- Prefer explicit session, frame, color-space, scale-factor, and path choices
  over defaults when deterministic cross-platform behavior matters.
- Set command-line switches before the `ready` event.
- Re-run signed, packaged smoke tests after changing ASAR integrity,
  notifications, TCC-sensitive features, native media capture, or installers.
- Open the topic references before implementing a migration; they retain the
  exact option names, backport notes, and operating-system qualifications.
