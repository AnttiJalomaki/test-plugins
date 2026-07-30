# Platform, Window, and Interop

## Pass window insets through embedded Compose

`AbstractComposeView.consumeWindowInsets` defaults to `false`. Compose adjusts
insets for the view's size and position so child views continue to receive
updates. Set it to `true` to retain consuming behavior (1.9.0).

## Recalculate descendant insets

Use `Modifier.recalculateWindowInsets()` when an ancestor positions content
without calling `consumeWindowInsets()` and descendants must apply
`insetsPadding` against recalculated values (1.8.0).

## Use window-inset rulers

The common `WindowInsetsRulers` API replaces `InsetsRulers`. Combine rulers
with `innermostOf()`, replace `rulersIgnoringVisibility` with `maximum`, and
read animation information through `WindowInsetsAnimation` and
`getAnimation()` (1.9.0).

The global `ComposeUiFlags.areWindowInsetsRulersEnabled` switch was replaced
by the per-view `ComposeView.disableWindowInsetsRulers()` API (1.11.0).

## Read container and window geometry

Use `LocalWindowInfo.current.containerSize` for the current content-container
size rather than deriving window size from configuration screen dimensions
(1.8.0).

`WindowInfo` exposes window size in dp. `WindowInsets.cutoutPath` exposes the
actual display-cutout outline for layouts that need its shape (1.10.0).

Inset-aware Material 2 and Material 3 components include `displayCutout` in
their default insets. Override the component inset when that automatic
avoidance is undesirable (`material3-1.4.0`).

## Provide host-default composition locals

`compositionLocalWithHostDefaultOf` defines a composition local whose fallback
comes from the hosting environment, such as an Android `View` tag.
`HostDefaultKey` is an interface. Public `HostDefaultProvider` and
`LocalHostDefaultProvider` allow a custom host to supply platform-specific
local values (1.11.0).

## Compose before a view is attached

`ComposeViewContext` lets a `ComposeView` compose before attachment to a view
hierarchy. Start it with
`AbstractComposeView.createComposition(composeViewContext)` (1.11.0).

## Select dialog and popup host windows

Android Compose dialogs accept a custom `windowToken`. Popups accept custom
`windowToken` and `windowType` values. `DialogProperties.windowType` also lets
a service show a Compose dialog in an overlay window (1.11.0).

## Interoperate with Android paint

The `androidx.compose.ui.graphics.NativePaint` typealias is deprecated. Use
`android.graphics.Paint` directly. Replace `Paint.asFrameworkPaint()` with
the `Paint.nativePaint` extension so common code does not expose an Android
platform type through a typealias (1.11.0).

## Convert packed colors explicitly

Compose packed colors are not directly comparable with Android `ColorLong`
values. Convert using `toColorLong()` and `fromColorLong()` (1.9.0).

## Use renamed Android constants

The Android-derived `UiModes` constants object is named `AndroidUiModes` from
1.10.0.

## Customize Android semantics extras

An Android-specific `SemanticsPropertyKey` factory can expose custom semantics
values through `AccessibilityNodeInfo.getExtras` (1.9.0).

## Know the multiplatform boundary

`androidx.compose.runtime:runtime` publishes desktop, iOS, and native support
through Google Maven. This does not make the rest of AndroidX Compose
multiplatform by implication (1.9.0).

The `runtime-rxjava2` and `runtime-rxjava3` artifacts are multiplatform and
include JVM support (1.10.0).
