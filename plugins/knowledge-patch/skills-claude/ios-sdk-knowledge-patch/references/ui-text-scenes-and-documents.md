# UI, Text, Scenes, and Documents

Use this reference for SwiftUI behavior, Writing Tools delegates, localization,
text layout, navigation and gesture semantics, TextKit, and screen resolution.

## Contents

- [SwiftUI Alerts, Environment, and Preference Closures](#swiftui-alerts-environment-and-preference-closures)
- [Writing Tools Multiple-Container Delegate](#writing-tools-multiple-container-delegate)
- [Plain-Text Replacement Attributes](#plain-text-replacement-attributes)
- [Localized Text Migration](#localized-text-migration)
- [Control-Size Ranges](#control-size-ranges)
- [Section Actions](#section-actions)
- [SDK-Linked Text Direction](#sdk-linked-text-direction)
- [Button-Style Picker Sizing](#button-style-picker-sizing)
- [NavigationLink Container Values](#navigationlink-container-values)
- [SwiftUI Gesture Priority](#swiftui-gesture-priority)
- [TextKit 2 List Markers](#textkit-2-list-markers)
- [`UIScreen.mainScreen` Deprecation](#uiscreenmainscreen-deprecation)

## SwiftUI Alerts, Environment, and Preference Closures

With the iOS 18.4 SDK (`18.4`), a view's `tint(_:)` overrides button tint inside
its alerts and confirmation dialogs. Verify destructive, cancel, and default
actions under the inherited tint rather than assuming system-default coloring.

Content updates in `NavigationStack` or `NavigationSplitView` invalidate the
environment even if no environment property changed. Avoid treating an
environment-dependent recomputation as proof that an environment value changed.

The `onPreferenceChange` closure no longer has to be `@Sendable`. It can access
main-actor-isolated state without the unnecessary sendability diagnostics
produced by the earlier signature.

## Writing Tools Multiple-Container Delegate

The optional `UIWritingToolsCoordinatorDelegate` methods for multiple-container
support are available in iOS 18.4 (`18.4`). Implement them when a coordinator's
context spans more than one content container.

`writingToolsCoordinator:requestsRangeInContextWithIdentifierForPoint:completion:`
supports asynchronous use of its completion block. Do not assume the delegate
must determine and return the range synchronously.

## Plain-Text Replacement Attributes

In iOS 18.5 (`18.5`), a writing-tools coordinator configured with
`resultOptions = [.plainText]` no longer strips attributes from proposed
replacement text. The resulting `NSAttributedString` carries the attributes the
delegate provided through
`writingToolsCoordinator(_:requestsContextsFor:completion:)`, including when
using the asynchronous forms.

If the consumer truly needs unstyled text, remove attributes explicitly rather
than relying on `.plainText` to discard them.

## Localized Text Migration

The version 26 SDK (`26.0`) deprecates interpolation of nonlocalized types into
`LocalizedStringResource`, `String(localized:)`, and
`AttributedString(localized:)`. Supply a localized value, or wrap an intentional
description explicitly:

```swift
String(describing: value)
```

SwiftUI also deprecates concatenating `Text` with `+`. Use `Text` interpolation
so localization can reorder the interpolated elements.

## Control-Size Ranges

In the version 26 SDK (`26.0`), `ControlSize` conforms to `Comparable`, and
`View.controlSize(_:)` can clamp the environment's control size to a range. Use
a range when a component should respect environmental sizing within supported
minimum and maximum bounds.

## Section Actions

Use `sectionActions(content:)` from the version 26 SDK (`26.0`) to attach actions
to a SwiftUI `Section`. Presentation is platform-specific: actions remain
trailing on macOS but become individual form rows on iOS and iPadOS. Test the
semantic association and layout on every supported platform.

## SDK-Linked Text Direction

For apps built with the version 26 SDK (`26.0`), `Text`, `TextEditor`, and
`TextField` infer each paragraph's base writing direction from its content.
Choose an override according to intent:

- Set `AttributedString.writingDirection` on each paragraph for explicit
  content-level direction.
- Apply `.writingDirection(strategy: .layoutBased)` when the interface layout
  direction should determine text direction.

TextKit 2 indentation in OS 26-linked apps also follows the resolved paragraph
direction. Binaries built with older SDKs retain UI-language-based direction in
several interfaces, so compare binaries built with each supported SDK when
investigating a behavior change.

## Button-Style Picker Sizing

In apps built with the iOS 26 or macOS 26 SDK (`26.0`), button-like `Picker`
styles use fitted sizing by default. Apply `buttonSizing(_:)` when the picker
should flex to fill its container.

## NavigationLink Container Values

With the version 26 SDK (`26.0`), `NavigationLink` produces a single view rather
than a view list in list contexts. A `ContainerValues` entry set inside the link
label or its `ButtonStyle` may therefore no longer escape to the surrounding
container.

Move `containerValue(_:_:)` outside the `NavigationLink` when the enclosing list
or container needs to read it.

## SwiftUI Gesture Priority

For apps built with an OS 26 SDK (`26.0`), choose the relationship between a
SwiftUI gesture and an existing UIKit or AppKit recognizer explicitly:

- Use `highPriorityGesture(_:isEnabled:)` when the SwiftUI gesture must run
  first.
- Use `simultaneousGesture(_:isEnabled:)` when both should have equal priority.

Do not rely on the earlier implicit recognizer ordering when native and SwiftUI
views are composed.

## TextKit 2 List Markers

The version 26 SDK (`26.0`) adds `includesTextListMarkers` to `NSTextList`,
`NSTextContentStorage`, and `NSWritingToolsCoordinator`. Use it to control
whether attributed-string paragraphs contain their list-marker text.

TextKit 2 omits marker text by default. UIKit has used that behavior since iOS
18, and AppKit adopts it with macOS 26. Keep the marker setting consistent when
moving attributed strings between editing, Writing Tools, and serialization
paths.

## `UIScreen.mainScreen` Deprecation

`UIScreen.mainScreen` is deprecated in iOS 26, tvOS 26, and visionOS 26
(`26.0`).
