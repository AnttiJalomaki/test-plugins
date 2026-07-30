# UI, Text, Scenes, and Documents

## SwiftUI environment and presentation

- A view's `tint(_:)` overrides button tint inside its alerts and confirmation dialogs (18.4).
- Updating `NavigationStack` or `NavigationSplitView` content invalidates the environment even when no environment property changed; avoid assuming those updates are isolated from environment-dependent work (18.4).
- `sectionActions(content:)` adds actions to a `Section`. Actions remain trailing on macOS but render as individual form rows on iOS and iPadOS (26.0).
- The data-item- and error-object-based `alert` and `confirmationDialog` modifiers back-deploy to iOS and tvOS 15, macOS 12, watchOS 8, and visionOS 1 (27.0-beta4).
- The `@Entry` macro warns when an environment entry stores a default class instance or closure. Treat the warning as a prompt to review shared mutable defaults, object lifetime, and closure capture (27.0-beta4).

## Controls, menus, and navigation

`ControlSize` conforms to `Comparable`, and `View.controlSize(_:)` can clamp the environment control size to a range (26.0).

Button-like `Picker` styles use fitted sizing by default in apps built with the iOS 26 or macOS 26 SDK. Apply `buttonSizing(_:)` when a picker should flex to fill its container (26.0).

With the new SDK, `NavigationLink` produces one view rather than a view list in list contexts. If a `ContainerValues` value must escape the link label or its `ButtonStyle`, move `containerValue(_:_:)` outside the link (26.0).

On the iPadOS 27 menu bar, SwiftUI hides symbol images for most menu items by default while leaving non-symbol images visible. Apply `.labelStyle(.titleAndIcon)` when a symbol must remain. A `LabeledContent` in a `Menu` maps its value to the native menu-item subtitle (27.0-beta4).

Use `toolbarMinimizationBehavior` instead of `toolbarMinimizeBehavior` (27.0-beta4).

## Gestures and text input

For SDK 26 builds, use `highPriorityGesture(_:isEnabled:)` when a SwiftUI gesture must precede an existing UIKit or AppKit recognizer. Use `simultaneousGesture(_:isEnabled:)` when it needs equal priority (26.0).

`TextInputBorderShape` and `textInputBorderShape(_:)` customize text-input borders. `.squareBorder` and `.roundedBorder` are soft-deprecated in favor of `.bordered` (27.0-beta4).

Applying `.textSelection(.enabled)` to `Text` enables interactive system selection in apps built with the iOS or iPadOS 27 SDK (27.0-beta4).

## Writing direction and list markers

In new-SDK builds, `Text`, `TextEditor`, and `TextField` infer base writing direction independently for each paragraph. Set `AttributedString.writingDirection` per paragraph or use `.writingDirection(strategy: .layoutBased)` when layout direction should drive the result (26.0).

TextKit 2 indentation in version 26-linked apps also follows resolved paragraph direction. Older-SDK binaries retain UI-language-based direction in several interfaces.

`NSTextList`, `NSTextContentStorage`, and `NSWritingToolsCoordinator` expose `includesTextListMarkers`. TextKit 2 omits markers from attributed-string paragraphs; UIKit has done so since iOS 18, and AppKit adopts this behavior with macOS 26 (26.0).

## Writing Tools

The optional `UIWritingToolsCoordinatorDelegate` methods for multiple containers are available on iOS 18.4. `writingToolsCoordinator:requestsRangeInContextWithIdentifierForPoint:completion:` supports asynchronous use of its completion block (18.4).

When `resultOptions = [.plainText]`, proposed replacement text retains the `NSAttributedString` attributes supplied by the delegate through `writingToolsCoordinator(_:requestsContextsFor:completion:)`, including with async forms (18.5).

## Documents and export

The `Document` protocol combines `ReadableDocument` and `WritableDocument` for ordinary read-write documents. `ReferenceFileDocument` and `FileDocument` are deprecated. URL-based readers and writers no longer need explicit `Source` or `Destination` aliases because both associated types default to `URL` (27.0-beta4).

`fileExporter(isPresented:documents:contentTypes:onCompletion:onCancellation:)` exports multiple `WritableDocument` values whose destination is `URL` in one dialog and returns their resulting URLs (27.0-beta4).

`FileWrapperDocumentWriter.makeFileWrapper` now passes `previous: FileWrapper?` as a second closure argument. Update existing closure signatures. A package document can mutate and return that wrapper to avoid rewriting unchanged children; a single-file document may ignore it (27.0-beta4).

## Scenes, screens, and external displays

Apps built with the iOS 27 SDK must adopt the scene-based lifecycle or they fail to launch (27.0-beta4).

`UIScreen.mainScreen` is deprecated on iOS 26, tvOS 26, and visionOS 26. Resolve screen state from the relevant scene or window instead (26.0).

Deprecated `UIApplication` status-bar accessors can return null or NaN in iOS 27 SDK builds. Read status-bar values from `UIWindowScene.statusBarManager` (27.0-beta4).

Use `.sceneAccessory` with `ExternalNonInteractiveAccessory` to place noninteractive content in an external-display scene in iOS 27 SDK builds (27.0-beta4).

## iPad and Mirroring beta checks

An iPad app built with the iOS 27 SDK is treated as not continuously resizable when `UISupportedInterfaceOrientations` omits an orientation. Until corrected, declare all four orientations in `Info.plist` to preserve continuous resizing (27.0-beta4).

Also test these beta regressions rather than coding around them as permanent behavior:

- iPhone Mirroring can ignore declared orientations or initially attach a scene to a non-main `UIScreen`.
- `UIRequiresFullScreen` scenes on iPad and in Mirroring can receive continuous resize updates rather than discrete screen changes.
- Full-screen iPad windows can mutate `UIScreen.main.bounds`.
- iPhone-only layouts can break while iPad is in an unsupported orientation.
- `UISceneClosureConfirmation` can fail to present its confirmation dialog.
