# Processes, security, and diagnostics

## Renderer and frame diagnostics

`WebFrameMain.collectJavaScriptCallStack()` can collect a JavaScript stack from
an unresponsive renderer since 34.0.0. Enable the Chromium feature before
readiness and opt the document into stack collection:

```js
app.commandLine.appendSwitch(
  'enable-features',
  'DocumentPolicyIncludeJSCallStacksInCrashReports',
);

webContents.on('unresponsive', async () => {
  console.log(await webContents.mainFrame.collectJavaScriptCallStack());
});
```

Serve the renderer with
`Document-Policy: include-js-call-stacks-in-crash-reports`.

`WebFrameMain.detached` identifies a frame that is unloading, while
`isDestroyed()` detects destruction. Since 34.0.0,
`webFrameMain.fromId(processId, frameId)` does not return an unloading frame
that fails to match the requested IDs.

Electron 38.0.0 deprecates `webFrame.routingId` and
`findFrameByRoutingId()`. Use renderer-side `frameToken` and
`findFrameByToken()`, then resolve from the main process with
`webFrameMain.fromFrameToken(processId, frameToken)`.

The `WebContents` `console-message` event deprecates positional `level`,
`message`, `line`, and `sourceId` arguments in 35.0.0. Read the event object;
`line` becomes `lineNumber`, `frame` is included, and `level` is `info`,
`warning`, `error`, or `debug`:

```js
webContents.on(
  'console-message',
  ({ level, message, lineNumber, sourceId, frame }) => {},
);
```

## Utility and child processes

- Utility processes emit `error` for V8 fatal errors and support diagnostic
  reports since 34.0.0.
- Since 37.0.0, an unhandled promise rejection warns instead of crashing a
  utility process. Install an `unhandledRejection` handler and call
  `process.exit(1)` to retain fail-fast behavior.
- `process.exit()` terminates utility processes synchronously since 37.0.0;
  pending output such as a preceding `console.log()` might not flush.
- Child-process exit details may report `memory-eviction` as the reason since
  40.0.0. Treat it as a recognized exit cause.
- On macOS, `utilityProcess` accepts `disclaim` for TCC disclaiming since
  41.0.0; the option is also available in Electron 39 and 40.

## ASAR integrity and packaging security

ASAR integrity is stable since 39.0.0. When enabled, Electron verifies
packaged `app.asar` against its build-time hash and forcefully terminates the
application if the hash is absent or mismatched. Electron Packager 19
separately enables ASAR packaging by default.

Since 41.0.0, macOS apps can embed a digest that protects the ASAR integrity
metadata itself. With `@electron/asar` 4.1.0 or later, enable the digest and
re-sign the app:

```sh
asar integrity-digest on /path/to/YourApp.app
```

## Runtime flags and instrumentation

Electron accepts Node.js flags `--no-experimental-global-navigator` and
`--experimental-network-inspection` since 37.0.0; both also shipped in
Electron 35 and 36.

Electron accepts `--experimental-transform-types` since 41.0.0, also present
in Electron 39 and 40. It accepts
`--experimental-inspector-network-resource` since 43.0.0.

Enable script attribution URLs for `long-animation-frame` entries with the
`AlwaysLogLOAFURL` feature (since 41.0.0, also in Electron 39 and 40):

```js
app.commandLine.appendSwitch('enable-features', 'AlwaysLogLOAFURL');
```

Since 42.0.0, `contentTracing` can collect heap profiles and renderer OOM
diagnostics can include a JavaScript stack.

## Security and operating-system controls

`safeStorage` gains asynchronous functionality and additional storage
backends in 42.0.0. Keep backend-dependent work asynchronous.

The `WasmTrapHandlers` fuse enables WebAssembly trap-handler support since
42.0.0. Treat fuse configuration as a packaging-time security decision.

Disable macOS geolocation with a command-line switch before readiness (since
41.0.0):

```js
app.commandLine.appendSwitch('disable-geolocation');
```

Renderer clipboard access is deprecated in 40.0.0 and the Electron renderer
module is removed in Electron 44. Use `navigator.clipboard` or a minimal
preload/context-bridge API rather than exposing the full Electron module.

On macOS, `process.getSystemMemoryInfo()` includes `fileBacked` and `purgeable`
since 38.0.0; the fields are also available in Electron 37.
