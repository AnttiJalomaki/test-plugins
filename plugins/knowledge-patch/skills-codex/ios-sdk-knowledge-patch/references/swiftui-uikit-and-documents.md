# SwiftUI, UIKit, and Documents

## SwiftUI behavior and layout

### Environment, tint, and preferences

- A view's `tint(_:)` overrides button tint inside its alerts and confirmation
  dialogs. Navigation stack and split-view content updates invalidate the
  environment even when no environment property changes (18.4).
- `onPreferenceChange` closures no longer need to be `@Sendable`, avoiding
  unnecessary diagnostics when accessing main-actor-isolated state (18.4).
- `ControlSize` conforms to `Comparable`, and `View.controlSize(_:)` can clamp
  the environment control size to a range (26.0).
- `@Entry` warns of potential issues when an environment entry's default is a
  class instance or closure (27.0-beta4).

### Sections, pickers, and menus

Use `sectionActions(content:)` to add actions to a `Section`. They stay trailing
on macOS but become separate form rows on iOS and iPadOS (26.0).

In binaries built with the iOS 26 or macOS 26 SDK, button-like `Picker` styles
fit their contents by default. Apply `buttonSizing(_:)` when they should flex to
fill the container (26.0).

On the iPadOS 27 menu bar, SwiftUI hides symbol images for most items while
leaving nonsymbol images visible. Apply `.labelStyle(.titleAndIcon)` to retain a
symbol. `LabeledContent` inside `Menu` maps its value to the native item subtitle
(27.0-beta4).

### Navigation and gestures

With the new SDKs, `NavigationLink` produces one view instead of a view list in
list contexts. If a `ContainerValues` value must escape the label or its
`ButtonStyle`, put `containerValue(_:_:)` outside the link (26.0).

For OS 26 SDK builds, use `highPriorityGesture(_:isEnabled:)` when a SwiftUI
gesture must precede an existing UIKit or AppKit recognizer, and
`simultaneousGesture(_:isEnabled:)` for equal priority (26.0).

## Text and writing direction

`Text`, `TextEditor`, and `TextField` infer each paragraph's base direction from
content in new-SDK builds. Set `AttributedString.writingDirection` per paragraph
or `.writingDirection(strategy: .layoutBased)` for layout-based behavior.
TextKit 2 indentation in OS 26-linked apps follows the resolved paragraph
direction; older-SDK binaries retain UI-language-based direction in several
interfaces (26.0).

`NSTextList`, `NSTextContentStorage`, and `NSWritingToolsCoordinator` expose
`includesTextListMarkers`. TextKit 2 omits marker text; UIKit has done so since
iOS 18, and AppKit adopts that behavior with macOS 26 (26.0).

In apps built with the iOS or iPadOS 27 SDK, applying
`.textSelection(.enabled)` to `Text` enables system interactive selection
(27.0-beta4).

Use `TextInputBorderShape` and `textInputBorderShape(_:)` for text-input
borders. `.squareBorder` and `.roundedBorder` are soft-deprecated in favor of
`.bordered` (27.0-beta4).

## Images, toolbars, alerts, and dialogs

`AsyncImage` automatically uses standard HTTP caching. Its new initializers can
take a `URLRequest` for a per-image cache policy, while
`asyncImageURLSession(_:)` supplies the session for descendant async images
(27.0-beta4).

Rename `toolbarMinimizeBehavior` uses to `toolbarMinimizationBehavior`. The
data-item- and error-object-based `alert` and `confirmationDialog` modifiers
back-deploy to iOS and tvOS 15, macOS 12, watchOS 8, and visionOS 1
(27.0-beta4).

## Writing Tools

Optional `UIWritingToolsCoordinatorDelegate` methods for multiple containers
are available. The
`writingToolsCoordinator:requestsRangeInContextWithIdentifierForPoint:completion:`
completion block can be completed asynchronously (18.4).

With `resultOptions = [.plainText]`, proposed replacement text retains the
`NSAttributedString` attributes originally supplied by
`writingToolsCoordinator(_:requestsContextsFor:completion:)`, including when
using its async forms (18.5).

## App Intents and Shortcuts

- The `notes.createNote` and `notes.updateNote` schemas accept `AttributedString`
  for `name`; `calendar.deleteEvents` is renamed to singular
  `calendar.deleteEvent` (27.0-beta4).
- Schema defaults might not apply to Set-valued parameters. Provide an explicit
  `@Parameter` default such as an empty set (27.0-beta4).
- Non-SF Symbol entity images might not appear in Siri. Workout-audio entities
  registered with `RelevantEntities` might not appear in the Fitness media
  picker (27.0-beta4).
- A shortcut whose app intent uses Duration or `LPLinkMetadata` might fail to
  edit using **Describe a change** (27.0-beta4).
- Adopters of the `createDraft`, `updateDraft`, `replyMail`, `forwardMail`,
  `message`, or `draft` email AssistantSchemas can fail to compile because the
  parameter types changed (26.6-rc).

## WebPage

`WebPage.load` APIs return an `AsyncSequence` of relevant navigation events.
`currentNavigationEvent` is removed; consume the indefinite `navigations`
sequence. A `WebPage` can load a URL directly, and HTML-string loading has a
default `baseURL` (26.6-rc).

## Documents and export

The `Document` protocol unifies `ReadableDocument` and `WritableDocument` for
ordinary read-write documents. `ReferenceFileDocument` and `FileDocument` are
deprecated. URL-based readers and writers can omit explicit `Source` and
`Destination` aliases because the associated types default to `URL`
(27.0-beta4).

`fileExporter(isPresented:documents:contentTypes:onCompletion:onCancellation:)`
exports a collection of `WritableDocument` values whose destination is `URL` in
one dialog and returns the output URLs (27.0-beta4).

`FileWrapperDocumentWriter.makeFileWrapper` closures receive a second
`previous: FileWrapper?` argument. Update existing closure signatures. Package
documents may mutate and return the previous wrapper to avoid rewriting
unchanged children; single-file documents may ignore it (27.0-beta4).

## UIKit scenes, screens, and windows

### Screen and status bar

`UIScreen.mainScreen` is deprecated on iOS 26, tvOS 26, and visionOS 26
(26.0). Deprecated `UIApplication` status-bar accessors can return null or NaN
in iOS 27 SDK builds; use `UIWindowScene.statusBarManager` instead.
`UISceneClosureConfirmation` might not present its dialog in the beta
(27.0-beta4).

### Lifecycle and external display

Apps built with the iOS 27 SDK must adopt the scene lifecycle or they fail to
launch (27.0-beta4). Such apps can put noninteractive content on an
external-display scene with `.sceneAccessory` and
`ExternalNonInteractiveAccessory` (27.0-beta4).

### iPad resizing and mirroring beta behavior

An iPad app built with the iOS 27 SDK is treated as not continuously resizable
if `UISupportedInterfaceOrientations` omits any orientation. Declare all four in
`Info.plist` until the beta defect is fixed (27.0-beta4).

iOS 27 SDK builds also have beta regressions around orientation and screen
identity: iPhone Mirroring can ignore declared orientations or initially attach
a scene to a non-main `UIScreen`; `UIRequiresFullScreen` scenes on iPad and in
Mirroring can receive continuous resize updates instead of discrete screen
changes. Full-screen iPad windows can mutate `UIScreen.main.bounds`, and
iPhone-only layouts can break while iPad is in an unsupported orientation
(27.0-beta4).

## Diagnostics and migration notes

In this beta, critical alerts are automatically enabled for every app that asks
for notification permission. Users who do not want them must turn off Critical
Alerts in the app's notification settings (27.0-beta4).

The original `MXMetricManager`, `MXMetricManagerSubscriber`, `MXMetricPayload`,
and `MXDiagnosticPayload` APIs are not recommended for new adoption; use
`MetricManager` (27.0-beta4).

On Demand Resources and `NSBundleResourceRequest` are deprecated. Move
downloadable app content to Background Assets (27.0-beta4).
