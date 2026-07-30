# Sessions, protocols, and extensions

## Preloads and service workers

Electron 35.0.0 deprecates `Session.setPreloads()` and `getPreloads()`. Use
per-script `registerPreloadScript()`, `unregisterPreloadScript()`, and
`getPreloadScripts()` so independent components do not replace a shared list.
Registration type may be `frame` or `service-worker`.

A service-worker preload uses `ipcRenderer`; communicate from the main process
through `ServiceWorkerMain.ipc`. `ServiceWorkers.startWorkerForScope()` starts
a worker deliberately, and `running-status-changed` observes its state.

Replace deprecated `session.serviceWorkers.fromVersionID(versionId)` with:

- `getInfoFromVersionID(versionId)` for the information record; or
- `getWorkerFromVersionID(versionId)` for the `ServiceWorkerMain` object.

Dynamic ESM `import()` is available in preload scripts with context isolation
disabled since 39.0.0; the feature was also released in Electron 37 and 38.

The experimental `contextBridge.executeInMainWorld(executionScript)` API can
evaluate a script in the main world across the bridge (since 35.0.0). Keep the
script and exposed surface narrowly scoped.

## Web-request filters and protocol handlers

An empty `WebRequestFilter.urls` array no longer matches everything. Since
35.0.0, use the explicit pattern:

```js
const filter = { urls: ['<all_urls>'] };
```

Since 36.0.0, `excludeUrls` removes matching patterns from a broader filter.

`net.request` accepts `bypassCustomProtocolHandlers` for requests that must
skip registered application handlers. Added in 40.0.0, it is also present in
Electron 38 and 39.

The deprecated `null` value for `ProtocolResponse.session` was removed in
37.0.0. Pass a session explicitly. To recreate a random independent session,
create a unique partition with `session.fromPartition(randomString)`, while
accounting for the overhead of single-purpose sessions.

Requests through `net` in utility processes can use an Electron session since
43.0.0.

`app.getApplicationInfoForProtocol()` is supported on Linux since 43.0.0.

## Extensions

Electron 36.0.0 moves extension methods and lifecycle events from `Session` to
`session.extensions`. Migrate `loadExtension()`, `removeExtension()`,
`getExtension()`, `getAllExtensions()`, and `extension-loaded`,
`extension-unloaded`, and `extension-ready` listeners to that object.

Electron 42.0.0 adds the `allowExtensions` custom-scheme privilege:

```js
protocol.registerSchemesAsPrivileged([
  {
    scheme: 'app',
    privileges: { standard: true, secure: true, allowExtensions: true },
  },
]);
```

This allows Chrome extensions to operate on the privileged application
protocol.

Since 43.0.0, `chrome.scripting.insertCSS()` and `removeCSS()` follow Chrome's
fallback-frame behavior. If an extension can access the creator page, CSS
injection may affect its `about:blank` or `data:` child frame. Narrow targets,
frame IDs, or match patterns if those frames must remain untouched.

## Storage, quotas, and downloads

Electron 36.0.0 removes the `syncable` quota type and deprecates the singular
`options.quota` property of `session.clearStorageData()`. `temporary` was the
only remaining quota type, so omit `quota` entirely.

Electron 42.0.0 removes the separate plural `options.quotas` object with its
upstream Chromium implementation. Do not pass it.

File System API grant status can persist within an Electron session since
39.0.0; the support was also released in Electron 37 and 38. Decide whether a
partition should retain grants when choosing persistent versus in-memory
session storage.

Since 43.0.0, file downloads default to the Downloads directory, falling back
to Home if Downloads does not exist.

## Authentication, permissions, and devices

Permission handling covers `document.executeCommand('paste')` since 35.0.0.
Treat it as a permission-controlled paste path rather than assuming a renderer
command bypasses policy.

WebSocket authentication is delivered through the `webContents` `login`
event. Added in 41.0.0, the behavior is also in Electron 39 and 40.

On macOS, Electron 42.0.0 adds Touch ID platform-authenticator configuration:

```js
app.configureWebAuthn({ touchID: { keychainAccessGroup } });
```

Use the session `select-webauthn-account` event to choose among discoverable
credentials.

Since 37.0.0, WebUSB and Web Serial enforce Chromium's specification-defined
device blocklists. Disable only when application policy requires it:

```js
app.commandLine.appendSwitch('disable-usb-blocklist');
app.commandLine.appendSwitch('disable-serial-blocklist');
```

WebUSB `USBDevice` objects expose `configurations` since 39.0.0.

## Cookies and shared dictionaries

Since 41.0.0, cookie `changed` events distinguish write outcomes:

- a new cookie reports `inserted`;
- deletion reports `explicit`;
- an identical reset reports `inserted-no-change-overwrite`;
- an attribute-only change that retains the value reports
  `inserted-no-value-change-overwrite`.

Since 34.0.0, `Session` can inspect and clear Brotli and Zstandard shared
compression dictionaries with `getSharedDictionaryUsageInfo()`,
`getSharedDictionaryInfo(options)`, `clearSharedDictionaryCache()`, and
isolation-key-scoped `clearSharedDictionaryCacheForIsolationKey(options)`.
