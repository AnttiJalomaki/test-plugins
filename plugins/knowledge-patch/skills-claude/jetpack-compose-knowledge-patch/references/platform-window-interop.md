# Platform, Window, and Interop

## Window size and geometry

### Content-container size (1.8.0)

Read the current content-container size from
`LocalWindowInfo.current.containerSize`. Do not derive window size from
configuration screen dimensions; lint warns about that pattern.

### Window geometry (1.10.0)

`WindowInfo` exposes window size in dp. `WindowInsets.cutoutPath` exposes the
actual display-cutout outline for layouts that need its shape rather than only
its insets.

## Window insets and rulers

### Recalculation after ancestor alignment (1.8.0)

Apply `Modifier.recalculateWindowInsets()` when descendants need to use
`insetsPadding` after an ancestor aligned them without calling
`consumeWindowInsets()`.

### ComposeView pass-through (1.9.0)

`AbstractComposeView.consumeWindowInsets` defaults to `false`. Compose
adjusts insets for its view's size and position while allowing child views to
receive updates. Set the property to `true` to retain consuming behavior.

### Common ruler API (1.9.0)

`WindowInsetsRulers` replaces `InsetsRulers`. Merge rulers with
`innermostOf()`, rename `rulersIgnoringVisibility` to `maximum`, and use
`WindowInsetsAnimation` with `getAnimation()` for animation data.

### Per-view ruler control (1.11.0)

Replace the `ComposeUiFlags.areWindowInsetsRulersEnabled` flag with
`ComposeView.disableWindowInsetsRulers()` when a particular view must opt out.

## Resources and fonts

### Configuration-aware resources (1.9.0)

Use `LocalResources.current` for Android resource access that must react to
configuration changes. The read invalidates composition so later lookups see
the new configuration.

### Resource-font failures (1.8.0)

A resource font that cannot load falls back silently to the default font
instead of throwing during measurement.

## Clipboard and tooltips

### Common APIs (1.8.0)

Foundation and UI provide a common `Clipboard` interface and its composition
local. `BasicTooltip` is available to common Foundation source sets.

## Android hosting

### Unattached ComposeView composition (1.11.0)

`ComposeViewContext` lets a `ComposeView` compose before attachment to a view
hierarchy. Start it with
`AbstractComposeView.createComposition(composeViewContext)`.

### Dialog and popup windows (1.11.0)

Android Compose dialogs accept a custom `windowToken`; popups accept custom
`windowToken` and `windowType`. `DialogProperties.windowType` also lets a
service display a Compose dialog in an overlay window.

## Android interop

### Paint (1.11.0)

The `androidx.compose.ui.graphics.NativePaint` typealias is deprecated. Use
`android.graphics.Paint` directly. Replace `Paint.asFrameworkPaint()` with
the `Paint.nativePaint` extension so common code does not expose an Android
platform type through a typealias.

### Android constants and parsing (1.10.0)

Rename the Android-derived `UiModes` constants object to `AndroidUiModes`.
`TextDirection`, `TextAlign`, `Hyphens`, and `FontSynthesis` `valueOf`
functions throw `IllegalArgumentException` for unknown values; validate or
handle invalid external strings rather than expecting a fallback.
