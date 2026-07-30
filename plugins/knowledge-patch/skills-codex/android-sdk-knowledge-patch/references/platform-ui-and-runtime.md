# Platform, UI, and Runtime Migration

## Runtime and language assumptions

### Fixed-rate scheduling

For an app targeting API 36, `ScheduledThreadPoolExecutor.scheduleAtFixedRate()` runs at most one missed invocation after the process returns to a valid lifecycle. It no longer immediately replays every missed invocation. Test code that depends on catch-up behavior; use the `STPE_SKIP_MULTIPLE_MISSED_PERIODIC_TASKS` compatibility flag to exercise the change. (`api-36`)

### App memory limits

Android 17 applies RAM-based limits to a subset of devices and to all apps, independent of target SDK. When the limiter terminates a process, `ApplicationExitInfo` reports `REASON_OTHER` and a description containing `MemoryLimiter:AnonSwap`.

- Register `ProfilingManager.TRIGGER_TYPE_ANOMALY` when a heap dump would help diagnose the exit.
- Inspect the limiter with `adb shell am memory-limiter status`.
- Exercise a process with `adb shell am memory-limiter manual <pid> <limit>`.
- Exempt a UID during testing with `adb shell am memory-limiter ignore <uid>`.

### Unsupported runtime internals

Apps targeting API 37 receive the lock-free `MessageQueue` implementation. Public APIs retain their behavior, but reflection against private fields or methods can break; remove or isolate such dependencies.

Android 17 also prevents API 37-targeted apps from mutating `static final` fields. Reflection throws `IllegalAccessException`, and JNI field setters crash the app. Replace test or framework code that depends on mutating constants. (`api-37`)

## Windowing, layout, and navigation

### Edge-to-edge

For an API 36-targeted app on Android 16, `windowOptOutEdgeToEdgeEnforcement` is deprecated and ignored. The opt-out still takes effect when the same app runs on Android 15. Remove it and make layouts consume or avoid system-bar insets correctly on both versions.

### Predictive back

Android 16 enables predictive-back system animations for API 36 targets. Legacy `onBackPressed` callbacks and `KEYCODE_BACK` dispatch no longer occur. Migrate to the supported back APIs. A temporary application- or activity-level opt-out remains available:

```xml
<application android:enableOnBackInvokedCallback="false" />
```

After migration, a long press of Back in 3-button navigation also shows predictive back.

### Large-screen restrictions

For API 36 targets on displays of at least `sw600dp`, Android 16 ignores requested orientation, `resizeableActivity`, minimum and maximum aspect ratios, and the related runtime APIs in full-screen and multi-window modes. Games and smaller displays are exempt. The temporary application- or activity-level opt-out below stops working when the app targets API 37:

```xml
<property android:name="android.window.PROPERTY_COMPAT_ALLOW_RESTRICTED_RESIZABILITY"
          android:value="true" />
```

Treat responsive large-screen layout as the durable migration.

### Configuration changes and IME state

Android 17 no longer recreates activities by default for keyboard, keyboard-hidden, navigation, touchscreen, color-mode, or desktop UI-mode transitions. If resource reloading depends on recreation, opt in with `android:recreateOnConfigChanges`; otherwise handle the change directly.

When the app does not handle a configuration change, Android 17 no longer restores the prior keyboard visibility after recreation. Use `windowSoftInputMode="stateAlwaysVisible"`, or explicitly request the IME from `onCreate()` or `onConfigurationChanged()`.

### Captured touchpads

During pointer capture, Android 17 converts touchpad motion and scrolling to captured mouse-style relative events. Software that needs raw absolute finger positions must call `View.requestPointerCapture(View.POINTER_CAPTURE_MODE_ABSOLUTE)`.

### Desktop pinned layer

In desktop mode, an app holding both `USE_PINNED_WINDOWING_LAYER` and picture-in-picture permission can request an interactive, always-on-top pinned window.

## Text, input, and accessibility

### Font metrics

For API 36 targets, `TextView`'s `elegantTextHeight` is deprecated and ignored, and compact variants of the affected UI fonts can no longer be selected. Recheck layouts using Arabic, Lao, Myanmar, Tamil, Gujarati, Kannada, Malayalam, Odia, Telugu, and Thai.

### Accessibility announcements

Android 16 deprecates `announceForAccessibility()` and `TYPE_ANNOUNCEMENT`. Use the semantic mechanism that matches the change:

- pane titles for structural changes;
- accessibility live regions for important dynamic content;
- error-specific accessibility events or `TextView.setError()` for validation failures.

### Complex IME composition

API 37 adds `TextAttribute.Builder.setTextSuggestionSelected()`, `TextAttribute.isTextSuggestionSelected()`, and `AccessibilityEvent.setTextChangeTypes()`/`getTextChangeTypes()`. CJKV IMEs, custom input connections, and accessibility services can use them to distinguish composition, candidate selection, and commit changes. Standard `TextView` behavior is automatic for API 37 targets.

### Password visibility

For API 37 targets, password visibility is split by input source. `show_passwords_physical` hides every character entered from a physical device by default, while touchscreen entry follows `show_passwords_touch`. Framework text fields adopt the settings automatically; custom fields should use `ShowSecretsSetting`.

## Components and system UI

### Ordered broadcasts

On Android 16, receiver priorities affect ordering only within the same application process, not across processes or apps. Priorities are clamped between `SYSTEM_LOW_PRIORITY + 1` and `SYSTEM_HIGH_PRIORITY - 1`. Use an explicit coordination mechanism when cross-process ordering matters.

### Launcher icons

Beginning with Android 16 QPR2, the launcher synthesizes a themed icon when an app omits one. Add a monochrome layer to the adaptive icon to control the result.

### Photo Picker layout

Android 17 adds `PhotoPickerUiCustomizationParams`, which can change the picker grid from square cells to a 9:16 portrait aspect ratio.

### Notifications and widgets

Android 17 enforces strict size limits on custom notification views. Compatibility-test every custom layout.

For external-display widgets, `RemoteViews.setViewPadding()` accepts complex DP/SP units. Use `OPTION_APPWIDGET_DISPLAY_ID` to obtain metrics for the display hosting the widget.

### MediaStore version tokens

For API 36-targeted apps, `MediaStore.getVersion()` returns an app-specific value. Treat it as an opaque change token; do not parse it or infer device information from it.
