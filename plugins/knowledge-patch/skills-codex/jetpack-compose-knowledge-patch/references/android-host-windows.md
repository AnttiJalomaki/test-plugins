# Android Hosts, Windows, and Insets

## Host services and common APIs

### Expanded haptic feedback (1.8.0)

`LocalHapticFeedback` supplies a default Android implementation when the device
vibrator reports support. Available feedback types include `Confirm`,
`ContextClick`, `GestureEnd`, `GestureThresholdActivate`, `Reject`,
`SegmentFrequentTick`, `SegmentTick`, `ToggleOn`, `ToggleOff`, and `VirtualKey`.

### Common clipboard and tooltip APIs (1.8.0)

Foundation and UI expose a common `Clipboard` interface through a composition
local. `BasicTooltip` is also available from common Foundation code, so shared
code need not reach through an Android-only clipboard or tooltip facade.

### Configuration-aware resources (1.9.0)

Read Android resources that must change with configuration from
`LocalResources.current`. Reading the local invalidates composition when the
configuration changes, so later lookups see the new resource configuration.

### Host-default composition locals (1.11.0)

`compositionLocalWithHostDefaultOf` defines a local whose fallback comes from
the host, such as an Android `View` tag. `HostDefaultKey` is an interface.
Custom hosts can supply values through the public `HostDefaultProvider` and
`LocalHostDefaultProvider` APIs.

## Compose views and attachment

### Window-size and layout-rectangle observation (1.8.0)

`Modifier.onLayoutRectChanged` observes bounds relative to the root, window, or
screen, with debounce and throttle controls. It is lower overhead than
`onGloballyPositioned` for this job. Read the current content-container size from
`LocalWindowInfo.current.containerSize`; lint warns against deriving window
size from configuration screen dimensions.

### Inset pass-through for `AbstractComposeView` (1.9.0)

`AbstractComposeView.consumeWindowInsets` defaults to `false`. Insets are
adjusted for the Compose view's size and position and passed to child views.
Set the property to `true` only to preserve the older consuming behavior.

### Window geometry (1.10.0)

`WindowInfo` exposes the window size in dp. `WindowInsets.cutoutPath` exposes
the actual display-cutout outline for layouts that need more than edge inset
distances.

### Unattached Android composition (1.11.0)

`ComposeViewContext` lets a `ComposeView` compose before attachment to a view
hierarchy. Start it explicitly:

```kotlin
composeView.createComposition(composeViewContext)
```

The method is declared on `AbstractComposeView`.

## Insets and rulers

### Recalculate descendant insets (1.8.0)

Apply `Modifier.recalculateWindowInsets()` when an ancestor changes alignment
without calling `consumeWindowInsets()` and descendants use `insetsPadding`.
This recalculates the values at the new location.

### Common rulers and inset animations (1.9.0)

Use common `WindowInsetsRulers` instead of `InsetsRulers`. Combine rulers with
`innermostOf()`. The former `rulersIgnoringVisibility` name is now `maximum`.
Read animation information through `WindowInsetsAnimation` and
`getAnimation()`.

### Per-view ruler control (1.11.0)

The global `ComposeUiFlags.areWindowInsetsRulersEnabled` flag was replaced by
`ComposeView.disableWindowInsetsRulers()`. Disable rulers on the specific view
whose host integration requires it.

## Windows, dialogs, and popups

### Dialog and popup window control (1.11.0)

Android Compose dialogs accept a custom `windowToken`. Popups accept custom
`windowToken` and `windowType` values. `DialogProperties.windowType` allows a
service to display a Compose dialog in an overlay window.

## Android graphics and constants

### Parsing and UI-mode constants (1.10.0)

`TextDirection.valueOf`, `TextAlign.valueOf`, `Hyphens.valueOf`, and
`FontSynthesis.valueOf` throw `IllegalArgumentException` for unknown values;
validate external strings or handle that exception. The Android-derived
`UiModes` object is renamed to `AndroidUiModes`.

### Paint interop (1.11.0)

The `androidx.compose.ui.graphics.NativePaint` typealias is deprecated. Use
`android.graphics.Paint` directly on Android. Replace `Paint.asFrameworkPaint()`
with `Paint.nativePaint`, avoiding an Android platform typealias in common code.

## Host integration checklist

- Test inset propagation when Compose and Views are nested in either direction.
- Distinguish content-container size, full window size, and screen coordinates.
- Re-test cutouts and safe areas after edge-to-edge layout changes.
- Scope ruler disabling to the affected `ComposeView`.
- Verify overlay window tokens, types, and permissions on the actual host.
- Use configuration-aware resource reads for locale, density, theme, and other
  changing resource qualifiers.
