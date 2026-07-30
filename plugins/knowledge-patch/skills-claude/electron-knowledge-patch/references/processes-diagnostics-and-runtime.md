# Processes, Diagnostics, and Runtime Behavior

## Renderer and frame lifecycle

Since 34.0.0, `WebFrameMain.detached` identifies a frame that is unloading,
whereas `WebFrameMain.isDestroyed()` identifies one already destroyed.
During unloading, `webFrameMain.fromId(processId, frameId)` no longer returns
a frame whose IDs do not match those requested.

To collect a JavaScript stack from an unresponsive renderer, enable
`DocumentPolicyIncludeJSCallStacksInCrashReports` before startup and serve the
renderer with
`Document-Policy: include-js-call-stacks-in-crash-reports`. Then call
`WebFrameMain.collectJavaScriptCallStack()`, commonly from
`webContents.mainFrame` in an `unresponsive` handler:

```js
app.commandLine.appendSwitch(
  'enable-features',
  'DocumentPolicyIncludeJSCallStacksInCrashReports',
);

webContents.on('unresponsive', async () => {
  console.log(await webContents.mainFrame.collectJavaScriptCallStack());
});
```

The `webFrame.routingId` and `webFrame.findFrameByRoutingId(routingId)` APIs
are deprecated since 38.0.0. Use `webFrame.frameToken` and
`webFrame.findFrameByToken(frameToken)` in the renderer. Resolve the token in
the main process with `webFrameMain.fromFrameToken(processId, frameToken)`.

Since 41.0.0, PDF documents render in the existing `WebContents`; they no
longer create a guest `WebContents`. Detect PDF resources by inspecting the
frame tree.

## Console and input-event payloads

Since 35.0.0, the positional `level`, `message`, `line`, and `sourceId`
arguments of the `WebContents` `console-message` event are deprecated. Read
`level`, `message`, `lineNumber`, `sourceId`, and `frame` from the event
object. `level` is `info`, `warning`, `error`, or `debug`.

```js
webContents.on(
  'console-message',
  ({ level, message, lineNumber, sourceId, frame }) => {},
);
```

The 37.0.0 notes record `WebContents` `before-mouse-event`, which lets a
listener intercept and prevent delivery; the event is also present in
Electron 36.

## Utility-process failures and exits

- Since 34.0.0, utility processes emit `error` and support diagnostic reports
  when V8 encounters a fatal error.
- Since 37.0.0, an unhandled promise rejection in a utility process warns
  instead of crashing. Recreate fail-fast behavior explicitly:

  ```js
  process.on('unhandledRejection', () => {
    process.exit(1);
  });
  ```

- Since 37.0.0, `process.exit()` terminates a utility process synchronously,
  matching Node.js. Pending output such as a preceding `console.log()` may not
  flush.
- Since 40.0.0, child-process exit details may report
  `"memory-eviction"` as the reason. Accept it in exit-reason handling.
- The 41.0.0 notes record `utilityProcess` `disclaim` for macOS TCC
  disclaiming; the option is also available in Electron 39 and 40.

## Node.js and command-line behavior

- Since 36.0.0, `app.commandLine` lowercases uppercase switches and
  arguments. It is for case-insensitive Chromium switches; read
  application-specific arguments from `process.argv`.
- The 37.0.0 notes record support for Node.js
  `--no-experimental-global-navigator` and
  `--experimental-network-inspection`; both flags are also available in
  Electron 35 and 36.
- The 41.0.0 notes record Node.js `--experimental-transform-types`; the flag
  is also available in Electron 39 and 40.
- Since 43.0.0, Electron passes through Node.js
  `--experimental-inspector-network-resource`.

## Performance, heap, and OOM diagnostics

- The 41.0.0 notes record script attribution for `long-animation-frame`
  entries through `AlwaysLogLOAFURL`; this is also available in Electron 39
  and 40:

  ```js
  app.commandLine.appendSwitch('enable-features', 'AlwaysLogLOAFURL');
  ```

- Since 42.0.0, `contentTracing` can collect heap profiles, and renderer
  out-of-memory diagnostics can capture a JavaScript stack trace.
- Since 42.0.0, the `WasmTrapHandlers` fuse can enable WebAssembly
  trap-handler support.

## Memory and cookie details

- The 38.0.0 notes record `fileBacked` and `purgeable` in macOS
  `process.getSystemMemoryInfo()`; these fields are also present in Electron
  37.
- Since 41.0.0, a cookie `changed` event reports `inserted` when setting a new
  cookie and `explicit` on deletion. Re-setting an identical cookie reports
  `inserted-no-change-overwrite`; changing only attributes without changing
  the value reports `inserted-no-value-change-overwrite`.
