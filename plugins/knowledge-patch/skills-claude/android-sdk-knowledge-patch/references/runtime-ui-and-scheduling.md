# Runtime, UI, and Scheduling

Use this reference for lifecycle, scheduling, windowing, text/input, and
framework-runtime changes. The parenthetical tags preserve the source batch
IDs `api-36` and `api-37`.

## Contents

- [Background work and execution](#background-work-and-execution)
- [Activity and window behavior](#activity-and-window-behavior)
- [Text, accessibility, and input](#text-accessibility-and-input)
- [Runtime integrity and resource pressure](#runtime-integrity-and-resource-pressure)
- [Widgets and media selection](#widgets-and-media-selection)

## Background work and execution

### Job quotas and abandoned work (`api-36`)

- Android 16 applies job runtime quotas to the active standby bucket, jobs
  started while visible that continue after visibility is lost, and jobs
  running alongside a foreground service. This affects `JobScheduler`,
  WorkManager, and DownloadManager.
- Prefer user-initiated data-transfer jobs where appropriate. Inspect stop
  reasons and, for jobs that have not run, call
  `JobScheduler.getPendingJobReasonsHistory()`.
- Keep `JobParameters` reachable until `jobFinished()`. If it is collected and
  the job later times out, the stop reason is
  `STOP_REASON_TIMEOUT_ABANDONED`; repeated abandonment can reduce scheduling
  frequency.
- `setImportantWhileForeground()` is ignored and
  `isImportantWhileForeground()` returns `false`.

### Missed fixed-rate executions (`api-36`)

For API 36-targeted apps, `scheduleAtFixedRate` runs at most one missed
invocation after the process returns to a valid lifecycle. It no longer replays
every missed run immediately. Test catch-up assumptions with the
`STPE_SKIP_MULTIPLE_MISSED_PERIODIC_TASKS` compatibility flag.

### Ordered broadcasts (`api-36`)

Receiver priority is honored only within one application process, not across
processes or apps. Priorities are constrained between
`SYSTEM_LOW_PRIORITY + 1` and `SYSTEM_HIGH_PRIORITY - 1`; use another
coordination mechanism when cross-process order matters.

### Exact idle alarms (`api-37`)

`AlarmManager.setExactAndAllowWhileIdle()` has an `OnAlarmListener` overload.
It supports an in-process exact callback without a `PendingIntent` and without
the associated long partial wakelock.

## Activity and window behavior

### Edge-to-edge and predictive back (`api-36`)

- For an API 36-targeted app on Android 16,
  `windowOptOutEdgeToEdgeEnforcement` is deprecated and ignored. The same
  opt-out can still work on Android 15, so remove it and make layouts consume
  insets on both releases.
- Predictive back system animations are enabled. Legacy `onBackPressed`
  callbacks and `KEYCODE_BACK` dispatch do not occur. Migrate to supported
  back APIs.
- A temporary application- or activity-level escape hatch is
  `android:enableOnBackInvokedCallback="false"`. Migrated apps also receive
  predictive back from a long press in 3-button navigation.

### Large-screen restrictions (`api-36`)

For API 36-targeted apps on displays of at least `sw600dp`, Android ignores
orientation requests, `resizeableActivity`, minimum and maximum aspect ratios,
and related runtime APIs in both full-screen and multi-window modes. Games and
smaller screens are exempt.

The following application- or activity-level opt-out is temporary and stops
working for API 37 targets:

```xml
<property android:name="android.window.PROPERTY_COMPAT_ALLOW_RESTRICTED_RESIZABILITY"
          android:value="true" />
```

### Configuration changes and IME restoration (`api-37`)

- Android 17 no longer recreates activities by default for keyboard,
  keyboard-hidden, navigation, touchscreen, color-mode, or desktop UI-mode
  transitions. If resource reload depends on recreation, opt in with
  `android:recreateOnConfigChanges`; otherwise update in place.
- When the app does not handle a configuration change, Android 17 no longer
  restores the keyboard's previous visibility. Use
  `windowSoftInputMode="stateAlwaysVisible"`, or request the IME explicitly
  from `onCreate()` or `onConfigurationChanged()`.

### Captured touchpads (`api-37`)

Pointer capture translates touchpad motion and scrolling into captured,
mouse-style relative events by default. Request raw absolute finger positions
with `View.requestPointerCapture(View.POINTER_CAPTURE_MODE_ABSOLUTE)`.

### Desktop pinned layer (`api-37`)

An app holding both `USE_PINNED_WINDOWING_LAYER` and picture-in-picture
permissions can request an interactive always-on-top pinned window in desktop
mode.

## Text, accessibility, and input

### Font metrics (`api-36`)

For API 36 targets, `TextView`'s `elegantTextHeight` is deprecated and ignored,
and compact variants of the affected UI fonts cannot be selected. Recheck
Arabic, Lao, Myanmar, Tamil, Gujarati, Kannada, Malayalam, Odia, Telugu, and
Thai layouts.

### Accessibility announcements (`api-36`)

Android 16 deprecates `announceForAccessibility()` and `TYPE_ANNOUNCEMENT`.
Represent structural changes with pane titles, important dynamic content with
accessibility live regions, and validation failures with error-specific events
or `TextView.setError()`.

### Themed launcher icons (`api-36`)

Starting in Android 16 QPR2, the launcher synthesizes a themed icon when an
app does not supply one. Add a monochrome layer to the adaptive icon to control
the result.

### Complex IME composition (`api-37`)

API 37 adds `TextAttribute.Builder.setTextSuggestionSelected()`,
`TextAttribute.isTextSuggestionSelected()`, and
`AccessibilityEvent.setTextChangeTypes()`/`getTextChangeTypes()`. CJKV IMEs,
custom input connections, and accessibility services can distinguish
composition, candidate selection, and commit changes. Standard `TextView`
handling is automatic for API 37-targeted apps.

### Password visibility (`api-37`)

For API 37 targets, `show_passwords_physical` hides every character entered
through a physical input device by default, while touchscreen entry follows
`show_passwords_touch`. Framework fields adopt the split automatically;
custom fields should use `ShowSecretsSetting`.

## Runtime integrity and resource pressure

### App memory limits (`api-37`)

Android 17 applies RAM-based limits on a subset of devices to all apps. A
limited process exits with `ApplicationExitInfo.REASON_OTHER` and
`MemoryLimiter:AnonSwap` in the description. `TRIGGER_TYPE_ANOMALY` can capture
a heap dump.

Inspect and exercise the mechanism with:

```shell
adb shell am memory-limiter status
adb shell am memory-limiter manual PID LIMIT
adb shell am memory-limiter ignore UID
```

### Message queue internals (`api-37`)

API 37-targeted apps receive a lock-free `MessageQueue` implementation.
Supported APIs preserve their behavior, but reflection against private fields
or methods can break and should be removed or isolated.

### Immutable constants (`api-37`)

Android 17 prevents API 37-targeted apps from changing `static final` fields.
Reflection throws `IllegalAccessException`; JNI field setters crash the app.

### Profiling and notifications (`api-37`)

`ProfilingManager` adds `COLD_START`, `OOM`, and
`KILL_EXCESSIVE_CPU_USAGE` triggers. Android 17 also enforces strict size
limits for custom notification views; test every custom layout.

## Widgets and media selection

### Display-aware widgets (`api-37`)

`RemoteViews.setViewPadding()` accepts complex DP/SP units. Read
`OPTION_APPWIDGET_DISPLAY_ID` to obtain metrics for the display hosting a
widget, especially on external displays.

### App-owned media under limited access (`api-36`)

For API 36-targeted apps on Android 16, the Photo Picker preselects app-owned
photos and videos when the user grants selected-media access. The user may
deselect those items, immediately revoking access despite ownership.
