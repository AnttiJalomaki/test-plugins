# Notifications, Security, and Packaging

## ASAR integrity

ASAR integrity is stable since 39.0.0. When enabled, Electron verifies
packaged `app.asar` against a build-time hash and terminates the application
when the hash is absent or mismatched. Electron Packager 19 separately enables
ASAR packaging by default.

Since 41.0.0, macOS bundles can embed a digest of their ASAR Integrity
metadata, validating the metadata itself at launch. Use `@electron/asar`
4.1.0 or later, then re-sign the application:

```bash
asar integrity-digest on /path/to/YourApp.app
```

## Clipboard isolation

Direct Electron `clipboard` API use in renderers is deprecated since 40.0.0.
Move privileged calls to a preload and expose only required operations through
`contextBridge`:

```js
const { clipboard, contextBridge } = require('electron');

contextBridge.exposeInMainWorld('clipboardAPI', {
  readText: () => clipboard.readText(),
});
```

Electron 44 removes Electron's `clipboard` module from renderer processes.
Use `navigator.clipboard` for ordinary access, retaining a narrow preload
bridge only for advanced Electron operations.

## Notifications

### Windows behavior

The 41.0.0 notes record a `reason` on the Windows `Notification` `closed`
event, identifying why it was dismissed, plus action buttons, select
dropdowns, and replies. Both additions are also available in Electron 40.

Since 42.0.0, Windows notifications accept `id`, `groupId`, `groupTitle`, and
`urgency`. `Notification.handleActivation(callback)` processes clicks,
replies, and action buttons even when a notification cold-starts the
application.

### macOS behavior

Since 42.0.0, macOS notifications use `UNNotification` rather than deprecated
`NSUserNotification`. The application must be code-signed for notifications
to display; an unsigned application's `Notification` emits `failed`.

`Notification.getHistory()` reads macOS notification history. Constructor
options `id` and `groupId` add custom identifiers and Notification Center
grouping.

Since 43.0.0, static macOS `Notification.remove()`, `removeAll()`, and
`removeGroup()` remove delivered notifications individually, in bulk, or by
group.

## Authentication, privacy, and storage

- Since 41.0.0, `--disable-geolocation` disables macOS location services:

  ```js
  app.commandLine.appendSwitch('disable-geolocation');
  ```

- The 41.0.0 notes record a macOS `utilityProcess` `disclaim` option for TCC
  disclaiming; the option is also present in Electron 39 and 40.
- Since 42.0.0,
  `app.configureWebAuthn({ touchID: { keychainAccessGroup } })` enables the
  macOS Touch ID platform authenticator. The session
  `select-webauthn-account` event selects among discoverable credentials.
- Since 42.0.0, asynchronous `safeStorage` functionality enables additional
  storage backends.

## Printing and auto-updates

- `PrinterInfo.isDefault` and `PrinterInfo.status` were removed in 35.0.0
  together with their upstream Chromium properties.
- Since 41.0.0, pass `usePrinterDefaultPageSize: true` to
  `webContents.print()` to use the printer's default page size:

  ```js
  webContents.print({ usePrinterDefaultPageSize: true });
  ```

- The 41.0.0 notes record MSIX support in `autoUpdater`. An update server can
  publish MSIX and Squirrel.Mac updates with essentially the same JSON
  response format. This support is also available in Electron 39.5.0 and
  40.2.0.
