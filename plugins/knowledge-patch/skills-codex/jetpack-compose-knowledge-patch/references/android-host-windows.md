# Android Hosts, Windows, and Insets

## Observe layout and window size

`Modifier.onLayoutRectChanged` in 1.8.0 reports root-, window-, or
screen-relative bounds with debounce and throttle controls. Prefer it over
`onGloballyPositioned` when repeated rectangle observation needs lower
overhead.

Read the current content-container size through
`LocalWindowInfo.current.containerSize`. Compose lint warns against deriving
window size from configuration screen dimensions.

`WindowInfo` exposes window size in dp as of 1.10.0.

## Recalculate descendant insets

`Modifier.recalculateWindowInsets()` in 1.8.0 lets descendants use
`insetsPadding` when an ancestor aligned them without calling
`consumeWindowInsets()`.

## ComposeView inset pass-through

`AbstractComposeView.consumeWindowInsets` defaults to `false` in 1.9.0. The
view automatically adjusts insets for its own size and position, allowing
child views to continue receiving updates. Set the property to `true` only to
retain the older consuming behavior.

## Window-inset rulers

The common `WindowInsetsRulers` API in 1.9.0 replaces `InsetsRulers`:

- Merge rulers with `innermostOf()`.
- Replace `rulersIgnoringVisibility` with `maximum`.
- Use `WindowInsetsAnimation` and `getAnimation()` for animation data.

In 1.11.0, the global `ComposeUiFlags.areWindowInsetsRulersEnabled` flag is
replaced by the per-view `ComposeView.disableWindowInsetsRulers()` API. Apply
the opt-out to the specific host view rather than setting a process-wide flag.

## Display cutout geometry

`WindowInsets.cutoutPath` in 1.10.0 exposes the display-cutout outline when a
layout needs the actual shape rather than rectangular inset distances.

Material 2 and Material 3 inset-aware components include `displayCutout` in
their default insets as of material3-1.4.0. Override the relevant component's
inset parameter if content should intentionally extend through the cutout
region.

## Configuration-aware Android resources

Use `LocalResources.current` for Android resources whose value must update
when configuration changes (since 1.9.0). Reading it invalidates composition,
so later resource lookups observe the new configuration.

## Host-default composition locals

`compositionLocalWithHostDefaultOf` in 1.11.0 defines a composition local whose
fallback can come from the host environment, such as a tag on an Android
`View`. `HostDefaultKey` is an interface. Custom hosts can provide platform
values through the public `HostDefaultProvider` and
`LocalHostDefaultProvider` APIs.

## Compose before view attachment

`ComposeViewContext` allows a `ComposeView` to compose before it is attached
to a view hierarchy (since 1.11.0). Start it explicitly:

```kotlin
composeView.createComposition(composeViewContext)
```

The declaring API is `AbstractComposeView.createComposition(composeViewContext)`.

## Dialog and popup window configuration

Android Compose dialogs in 1.11.0 accept a custom `windowToken`. Popups accept
custom `windowToken` and `windowType` values. `DialogProperties.windowType`
also allows a service to display a Compose dialog in an overlay window. Use
tokens and types that the hosting Android context is authorized to attach.

## Pointer-driven host focus

Mouse or touchpad presses outside the focused node clear focus by default in
1.10.0. Set
`AbstractComposeView.isClearFocusOnPointerDownEnabled = false` on a host that
must preserve the prior behavior.

## Android graphics and constants

The `androidx.compose.ui.graphics.NativePaint` typealias is deprecated in
1.11.0. Use `android.graphics.Paint` directly. Replace
`Paint.asFrameworkPaint()` with the `Paint.nativePaint` extension so common
code does not publicly expose an Android type through a typealias.

The Android-derived `UiModes` constants object is renamed to `AndroidUiModes`
in 1.10.0.
