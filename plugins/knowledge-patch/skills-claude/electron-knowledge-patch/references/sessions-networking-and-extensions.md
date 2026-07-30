# Sessions, Networking, and Extensions

## Session caches and preload registration

Since 34.0.0, a `Session` can inspect and clear Brotli and Zstandard shared
compression dictionaries with:

- `getSharedDictionaryUsageInfo()`
- `getSharedDictionaryInfo(options)`
- `clearSharedDictionaryCache()`
- `clearSharedDictionaryCacheForIsolationKey(options)` for one isolation key

Since 35.0.0, `Session.setPreloads()` and `getPreloads()` are deprecated.
Register scripts independently with `registerPreloadScript()`, remove them
with `unregisterPreloadScript()`, and enumerate them with
`getPreloadScripts()`. A registration can target `frame` or `service-worker`
contexts, avoiding whole-list replacement between libraries.

Attached service-worker preloads use `ipcRenderer`; main-process code
communicates through `ServiceWorkerMain.ipc`. Service-worker support also
provides `ServiceWorkers.startWorkerForScope()` and the
`running-status-changed` event.

`session.serviceWorkers.fromVersionID(versionId)` is deprecated. Use
`getInfoFromVersionID(versionId)` for the information record or
`getWorkerFromVersionID(versionId)` for its `ServiceWorkerMain`.

## Web-request filters and authentication

- Since 35.0.0, an empty `WebRequestFilter.urls` does not match all URLs. Use
  `{ urls: ['<all_urls>'] }` explicitly.
- Since 36.0.0, `WebRequestFilter.excludeUrls` removes matching URL patterns
  from a filter.
- The 41.0.0 notes record WebSocket authentication through the `webContents`
  `login` event; the support is also available in Electron 39 and 40.
- Chromium is deprecating `--host-rules`; use `--host-resolver-rules` from
  Electron 39 onward.

## Extensions and custom protocols

Since 36.0.0, extension methods and events live on `session.extensions`.
Replace `session.loadExtension()`, `session.removeExtension()`,
`session.getExtension()`, and `session.getAllExtensions()` with the
corresponding `session.extensions` methods. Move `extension-loaded`,
`extension-unloaded`, and `extension-ready` listeners to that `Extensions`
object as well.

Since 42.0.0, the `allowExtensions` privilege allows Chrome extensions to run
on a privileged custom protocol:

```js
protocol.registerSchemesAsPrivileged([
  {
    scheme: 'app',
    privileges: { standard: true, secure: true, allowExtensions: true },
  },
]);
```

Since 43.0.0, extension `chrome.scripting.insertCSS()` and `removeCSS()` match
Chrome for fallback frames such as `about:blank` and `data:`. If the extension
can access the page that created a fallback frame, injection can affect that
frame too. Narrow targets, frame IDs, or match patterns when such frames must
be excluded.

## Protocol and request sessions

- Since 37.0.0, `ProtocolResponse.session` cannot be `null`. To reproduce the
  former random independent-session behavior, create a unique partition with
  `session.fromPartition(randomString)` and assign it. Avoid unnecessary
  single-purpose sessions because they add overhead.
- The 40.0.0 notes record `net.request`'s
  `bypassCustomProtocolHandlers` option for skipping registered handlers; the
  option is also present in Electron 38 and 39.
- Since 43.0.0, `net` requests made from a utility process can use an Electron
  session.
- Since 43.0.0, `app.getApplicationInfoForProtocol()` works on Linux.

## Storage and file-system grants

- In 36.0.0, `syncable` was removed from the quota types accepted by
  `session.clearStorageData(options)`, and `options.quota` was deprecated.
  Because `temporary` was the only remaining quota type, omit `quota`.
- In 42.0.0, the upstream `quotas` object was removed from
  `Session.clearStorageData(options)`; do not pass `options.quotas`.
- The 39.0.0 notes record persistent File System API grant status within an
  Electron session; the support is also available in Electron 37 and 38.
- Since 43.0.0, file downloads default to Downloads, falling back to Home
  when the Downloads directory does not exist.

## USB and serial devices

- Since 37.0.0, WebUSB and Web Serial enforce Chromium's
  specification-defined device blocklists. Disable them only when required:

  ```js
  app.commandLine.appendSwitch('disable-usb-blocklist');
  app.commandLine.appendSwitch('disable-serial-blocklist');
  ```

- Since 39.0.0, Electron WebUSB `USBDevice` objects expose
  `configurations`.

## Permissions and renderer bridging

- Permission handling covers `document.executeCommand("paste")` since
  35.0.0.
- The experimental `contextBridge.executeInMainWorld(executionScript)` API
  introduced in 35.0.0 evaluates JavaScript in the main world across the
  context bridge.
- The 39.0.0 notes record dynamic `import()` in preload scripts when context
  isolation is disabled; this behavior is also available in Electron 37 and
  38.
