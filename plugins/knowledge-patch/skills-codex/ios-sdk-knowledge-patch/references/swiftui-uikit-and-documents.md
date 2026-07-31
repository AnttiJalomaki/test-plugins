# SwiftUI, UIKit, and Documents

## Preserve localization semantics

Interpolating a nonlocalized type into `LocalizedStringResource`,
`String(localized:)`, or `AttributedString(localized:)` produces a deprecation
warning with the iOS 26 SDK. Supply a localized value, or explicitly wrap an
intentional description with `String(describing:)`. (26.0)

SwiftUI also deprecates concatenating `Text` values with `+`. Use `Text`
interpolation so a translation can reorder the inserted content. (26.0)

## Account for corrected SwiftUI updates

A view's `tint(_:)` overrides the tint of buttons inside its alerts and
confirmation dialogs. `NavigationStack` and `NavigationSplitView` content
updates invalidate the environment even if no environment property changed.
Retest code that relied on stale environment-derived values. (18.4)

The `onPreferenceChange` closure no longer has to be `@Sendable`, which avoids
unnecessary diagnostics when it accesses main-actor-isolated state. Do not add
unsafe sendability workarounds to silence the old diagnostic. (18.4)

## Adopt Writing Tools multiple-container and async behavior

The optional `UIWritingToolsCoordinatorDelegate` methods for multiple-container
support are available in iOS 18.4. The
`writingToolsCoordinator:requestsRangeInContextWithIdentifierForPoint:completion:`
delegate method supports asynchronous use of its completion block, so retain
the context needed to finish the request rather than assuming synchronous
completion. (18.4)

When a writing-tools coordinator uses `resultOptions = [.plainText]`, proposed
replacement text retains the `NSAttributedString` attributes supplied by the
delegate from
`writingToolsCoordinator(_:requestsContextsFor:completion:)`. The same applies
to the async forms. Do not assume plain-text results arrive stripped of those
attributes. (18.5)

## Constrain and place controls deliberately

`ControlSize` conforms to `Comparable`, and `View.controlSize(_:)` can clamp
the environment's control size to a range. Use a range when a component may
adapt but must remain within supported bounds. (26.0)

Use `sectionActions(content:)` to attach actions to a `Section`. They remain
trailing on macOS, while iOS and iPadOS present them as individual form rows;
check the platform-specific layout instead of assuming header placement.
(26.0)

Button-like `Picker` styles use fitted sizing by default in apps built with the
iOS 26 or macOS 26 SDK. Apply `buttonSizing(_:)` when the picker should expand
to fill its container. (26.0)

## Adapt to SDK-linked paragraph direction

`Text`, `TextEditor`, and `TextField` infer each paragraph's base writing
direction from its content in new-SDK builds. Set
`AttributedString.writingDirection` per paragraph when the content needs an
explicit direction, or apply `.writingDirection(strategy: .layoutBased)` when
the surrounding layout should determine it. (26.0)

TextKit 2 indentation in OS 26-linked apps likewise follows resolved paragraph
direction. Older-SDK binaries retain UI-language-based direction in several
interfaces, so test old and newly linked binaries separately. (26.0)

## Move container values outside NavigationLink

With the new SDKs, `NavigationLink` produces one view rather than a view list in
list contexts. A `ContainerValues` value attached inside the link label or its
`ButtonStyle` may no longer escape the link. Move `containerValue(_:_:)`
outside the `NavigationLink` when the surrounding container needs it. (26.0)

## Set SwiftUI gesture precedence explicitly

For OS 26 SDK builds, attach `highPriorityGesture(_:isEnabled:)` when a SwiftUI
gesture must run before an existing UIKit or AppKit recognizer. Use
`simultaneousGesture(_:isEnabled:)` when the SwiftUI and native recognizers
should have equal priority. (26.0)

## Choose TextKit 2 list-marker representation

`NSTextList`, `NSTextContentStorage`, and `NSWritingToolsCoordinator` provide
`includesTextListMarkers` to control whether attributed-string paragraphs
contain marker text. TextKit 2 omits markers by default; UIKit has used that
behavior since iOS 18, and AppKit adopts it with macOS 26. Set the property
when interchange or editing code needs explicit marker content. (26.0)

## Replace global main-screen access

`UIScreen.mainScreen` is formally deprecated in iOS 26, tvOS 26, and visionOS
26. Derive the screen from the relevant `UIWindowScene`, window, view, or other
operation context instead of assuming one process-wide main display. (26.0)
