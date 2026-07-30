# Windows, Input, and Platform UI

## Window creation, sizing, and navigation

- The 37.0.0 notes record `window.open()` support for `innerWidth` and
  `innerHeight` in its feature string; the support is also present in Electron
  35 and 36.
- Since 39.0.0, `window.open()` popups are always resizable. Restore
  feature-controlled resizing through `setWindowOpenHandler()`:

  ```js
  webContents.setWindowOpenHandler((details) => ({
    action: 'allow',
    overrideBrowserWindowOptions: {
      resizable: details.features.includes('resizable=yes'),
    },
  }));
  ```

- Since 35.0.0, `webContents.navigationHistory.restore(index, entries)`
  restores a supplied history and selects the requested entry.
- The 41.0.0 notes record `webPreferences.focusOnNavigation: false`, which
  prevents a `WebContents` from receiving focus automatically during
  navigation; the option is also present in Electron 40.
- On Linux since 37.0.0,
  `BrowserWindow#isVisibleOnAllWorkspaces()` returns `false` when the window
  is not currently visible.

## Frameless windows and platform decorations

- Since 35.0.0, the `BrowserWindow` `roundedCorners` option works on Windows.
  `BrowserWindow.setVibrancy()` also accepts an optional animation parameter.
- Since 41.0.0, Wayland frameless windows have drop shadows and extended
  resize boundaries. Construct the window with `hasShadow: false` when no
  decoration is wanted.
- Since 43.0.0, Linux frameless windows have rounded corners by default. Set
  `roundedCorners: false` to disable them.
- Since 43.0.0, Linux Window Controls Overlay follows the native title-bar
  layout and user settings, so controls and their side vary. Keep content in
  the available title-bar area:

  ```css
  .titlebar-content {
    left: env(titlebar-area-x, 0px);
    width: env(titlebar-area-width, 100%);
  }
  ```

## Wayland, X11, GTK, and desktop environment

- Since 38.0.0, `ELECTRON_OZONE_PLATFORM_HINT` is removed and Chromium's
  `--ozone-platform` defaults to `auto`. Electron runs natively on Wayland
  in a Wayland session; pass `--ozone-platform=x11` for Xwayland.
- Since 36.0.0, Electron defaults to GTK 4 on GNOME. If loading GTK 2/3
  symbols causes a version-coexistence failure, use either startup form:

  ```sh
  electron --gtk-version=3
  ```

  ```js
  app.commandLine.appendSwitch('gtk-version', '3');
  ```

  Append the switch before readiness.
- The 37.0.0 notes record Linux X11 support for
  `screen.dipToScreenPoint(point)` and `screen.screenToDipPoint(point)`; this
  was also released in Electron 35 and 36.
- Since Electron 38, `XDG_CURRENT_DESKTOP` contains the real desktop
  environment instead of being overwritten with `Unity`;
  `ORIGINAL_XDG_CURRENT_DESKTOP` is removed.

## Menus, context menus, and shortcuts

- On Windows since 34.0.0, entering fullscreen hides the menu bar, matching
  Linux. The change was announced for Electron 33 but first shipped in 34.
- Since 36.0.0, `WebContents.focusedFrame` identifies the focused frame.
  Passing it as `Menu.popup()`'s `frame` option enables macOS Writing Tools,
  Autofill, Services, and other system integrations:

  ```js
  const window = BrowserWindow.getFocusedWindow();
  menu.popup({ window, frame: window.webContents.focusedFrame });
  ```

- The `system-context-menu` event works on Linux since 36.0.0.
- Since 37.0.0, macOS 14.4 and later support menu sublabels. macOS menus also
  support `palette` and `header` roles.
- Since 35.0.0, launch with
  `--enable-features=GlobalShortcutsPortal` to use the desktop portal's global
  shortcut implementation.
- Since 43.0.0, `globalShortcut.setSuspended()` suspends or resumes global
  shortcut handling, and `globalShortcut.isSuspended()` queries the state.
- Since 43.0.0, `MenuItem` constructor options and properties accept
  `accessibilityLabel` for a screen-reader-friendly label.

## Appearance and accessibility

- Since 36.0.0,
  `nativeTheme.shouldUseDarkColorsForSystemIntegratedUI` reports the
  operating system's integrated-UI appearance separately from the
  application's selected theme.
- The 38.0.0 notes record custom system accent colors and active-window border
  highlighting; this support is also present in Electron 37.
- The 40.0.0 notes record `window.setAccentColor(null)` for resuming the
  system accent after setting a custom one; this also works in Electron 38 and
  39.
- Since 42.0.0 on macOS,
  `nativeTheme.shouldDifferentiateWithoutColor` reflects the accessibility
  preference to distinguish content without relying on color.
- Since 35.0.0, Electron sets
  `NSPrefersDisplaySafeAreaCompatibilityMode` to `false` in `Info.plist`,
  removing the “Scale to fit below built-in camera” application option.

## Dialogs and file locations

- Since 38.0.0, `dialog.showMessageDialog()` with a parent centers on that
  parent instead of the monitor.
- In 41.0.0, Linux `showHiddenFiles` was deprecated for dialogs while
  remaining supported on macOS and Windows. It is removed on Linux in
  43.0.0.
- In Electron 43, an open/save dialog without `defaultPath` starts in
  Downloads, or Home when Downloads does not exist, rather than restoring the
  OS's last-used directory. Persist the chosen directory and pass it:

  ```js
  const { dialog } = require('electron');
  const path = require('node:path');

  let lastUsedPath;
  async function chooseFile() {
    const result = await dialog.showOpenDialog({ defaultPath: lastUsedPath });
    if (!result.canceled && result.filePaths.length) {
      lastUsedPath = path.dirname(result.filePaths[0]);
    }
    return result;
  }
  ```

- Electron 35 accepts file-dialog portal version 3, but `defaultPath` is
  unavailable on portal implementations older than version 4. When the path
  is required, start with `--xdg-portal-required-version=4`.

## Application and operating-system integration

- Since 35.0.0 on Windows, `query-session-end` is available and existing
  `session-end` events are improved.
- The 38.0.0 notes record the macOS `Tray` constructor's `guid`, which allows
  the icon to keep its position between launches; this also works in Electron
  37.
- The 38.0.0 notes record `app.getRecentDocuments()` on Windows and macOS;
  the support is also present in Electron 37.
- The 38.0.0 notes record `app.getPath('assets')` for the assets and resources
  location; this is also present in Electron 37.
- Since 35.0.0, `systemPreferences.isAeroGlassEnabled()` is deprecated
  without replacement. It always returned `true` from Electron 23 onward and
  is removed in Electron 36; delete conditional branches based on it.
