# Windows, menus, and platform integration

## Window creation, sizing, and appearance

- Windows fullscreen mode hides the menu bar since 34.0.0, matching Linux.
  This was announced earlier but first shipped in Electron 34.
- `BrowserWindow` supports `roundedCorners` on Windows, and
  `setVibrancy()` accepts an optional animation argument since 35.0.0.
- `window.open()` recognizes `innerWidth` and `innerHeight` since 37.0.0; the
  support was also released in Electron 35 and 36.
- Since 39.0.0, `window.open()` always creates resizable popups. Preserve a
  feature-string policy with a window-open handler:

```js
webContents.setWindowOpenHandler((details) => ({
  action: 'allow',
  overrideBrowserWindowOptions: {
    resizable: details.features.includes('resizable=yes'),
  },
}));
```

- `WebContents` emits `before-mouse-event` before delivering mouse input,
  allowing prevention since 37.0.0; it was also released in Electron 36.
- Set `webPreferences.focusOnNavigation` to `false` to prevent automatic focus
  during navigation. Added in 41.0.0, this is also in Electron 40.
- Frameless Wayland windows gain drop shadows and extended resize boundaries
  in 41.0.0. Use constructor option `hasShadow: false` to remove decorations.
- Frameless Linux windows have rounded corners by default since 43.0.0. Set
  `roundedCorners: false` to opt out.

Frameless Linux Window Controls Overlay follows native button availability,
side, and title-bar layout since 43.0.0. Do not reserve a fixed control area:

```css
.titlebar-content {
  left: env(titlebar-area-x, 0px);
  width: env(titlebar-area-width, 100%);
}
```

Electron can customize the system accent color and active-window border since
38.0.0, also available in Electron 37. After customization,
`window.setAccentColor(null)` resumes the system color since 40.0.0, also
available in Electron 38 and 39.

The custom `-electron-corner-smoothing` CSS property landed in Electron 36 and
is documented in 37.0.0. It accepts `0%` through `100%`; `system-ui` resolves
to 60% on macOS and 0% elsewhere. It affects corners, borders, outlines, and
shadows:

```css
.box {
  border-radius: 24px;
  -electron-corner-smoothing: system-ui;
}
```

## Menus, shortcuts, and input

`WebContents.focusedFrame` identifies the focused frame. On macOS, pass it as
`Menu.popup()`'s `frame` option to enable Writing Tools, Autofill, Services,
and other focused-frame integrations (since 36.0.0):

```js
menu.popup({
  window: BrowserWindow.getFocusedWindow(),
  frame: BrowserWindow.getFocusedWindow().webContents.focusedFrame,
});
```

Linux supports the `system-context-menu` event since 36.0.0. macOS 14.4 and
later support menu sublabels, and macOS menus accept `palette` and `header`
roles since 37.0.0.

`MenuItem` accepts and exposes `accessibilityLabel` for a screen-reader-friendly
label since 43.0.0.

Enable portal-backed global shortcuts with
`--enable-features=GlobalShortcutsPortal` (since 35.0.0).
`globalShortcut.setSuspended()` and `isSuspended()` suspend, resume, and query
global shortcut handling since 43.0.0.

## Navigation, frames, and PDFs

Restore supplied history and its selected entry with
`webContents.navigationHistory.restore(index, entries)` since 35.0.0.

PDF resources no longer create a separate guest `WebContents` since 41.0.0.
Inspect the existing contents' frame tree when detecting a PDF.

`screen.dipToScreenPoint()` and `screen.screenToDipPoint()` work on Linux X11
since 37.0.0, also released in Electron 35 and 36.
`BrowserWindow.isVisibleOnAllWorkspaces()` returns `false` on Linux when the
window is not currently visible since 37.0.0.

## Dialogs, printing, and filesystem-facing UI

- A parented `dialog.showMessageDialog()` centers on its parent rather than the
  monitor since 38.0.0.
- `webContents.print({ usePrinterDefaultPageSize: true })` uses the printer's
  default page size since 41.0.0.
- `PrinterInfo.isDefault` and `PrinterInfo.status` were removed in 35.0.0 with
  their Chromium counterparts.
- Linux deprecated `showHiddenFiles` in 41.0.0 and stopped supporting it in
  43.0.0. It remains supported on macOS and Windows.
- Since 43.0.0, open/save dialogs with no `defaultPath` start in Downloads, or
  Home when Downloads is absent, rather than restoring the last-used OS path.
  Persist the chosen directory and pass it as `defaultPath` if needed.
- Electron 35 supports file-dialog portals as old as version 3, but portal
  backends before version 4 ignore `defaultPath`. Require version 4 when the
  initial directory is essential:

```sh
electron --xdg-portal-required-version=4 .
```

## Notifications and session-ending behavior

Windows adds `query-session-end` and improves `session-end` events in 35.0.0.

Windows `Notification` `closed` events include a dismissal `reason` since
41.0.0. Actions support buttons, select dropdowns, and replies; both features
are also in Electron 40.

Since 42.0.0, macOS notifications use `UNNotification`; applications must be
code-signed or `Notification` emits `failed`. `Notification.getHistory()`
reads macOS history, and constructor `id` and `groupId` provide stable identity
and Notification Center grouping.

Windows notifications add `id`, `groupId`, `groupTitle`, and `urgency` in
42.0.0. `Notification.handleActivation(callback)` handles clicks, replies, and
action buttons even when activation cold-starts the app.

On macOS, `Notification.remove()`, `removeAll()`, and `removeGroup()` remove
delivered notifications individually, in bulk, or by group since 43.0.0.

## Desktop and accessibility integration

- Electron's `Info.plist` sets `NSPrefersDisplaySafeAreaCompatibilityMode` to
  `false` since 35.0.0, removing “Scale to fit below built-in camera.”
- `nativeTheme.shouldUseDarkColorsForSystemIntegratedUI` exposes the OS
  integrated-UI dark appearance separately from the application's selected
  theme since 36.0.0.
- `nativeTheme.shouldDifferentiateWithoutColor` exposes the macOS accessibility
  preference since 42.0.0.
- `Tray` accepts `guid` on macOS to retain tray position across launches since
  38.0.0, also available in Electron 37.
- `app.getRecentDocuments()` works on Windows and macOS since 38.0.0, also
  available in Electron 37.
- `app.getPath('assets')` exposes the assets/resources directory since 38.0.0,
  also available in Electron 37.
