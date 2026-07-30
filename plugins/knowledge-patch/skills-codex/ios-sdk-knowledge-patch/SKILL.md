---
name: ios-sdk-knowledge-patch
description: iOS SDK
version: iOS 27 beta 4 / Xcode 27 beta 4
license: MIT
metadata:
  author: Nevaberry
---


# iOS SDK Knowledge Patch

Use this skill when building, upgrading, testing, or distributing apps with recent
iOS SDKs and Xcode toolchains. Inspect the target SDK, Xcode version, deployment
targets, linked-on behavior, and device OS before applying version-dependent advice.

## Reference index

| Reference | Topics |
| --- | --- |
| [toolchain-build-and-testing.md](references/toolchain-build-and-testing.md) | Xcode hosts and SDKs, device floors, modules, Interface Builder, package builds, diagnostics, Simulator, testing |
| [networking-security-and-distribution.md](references/networking-security-and-distribution.md) | TLS, URLSession, VPN, entitlements, attribution, passkeys, enterprise and alternate distribution, App Store uploads |
| [persistence-foundation-and-commerce.md](references/persistence-foundation-and-commerce.md) | Core Data, StoreKit, localization, ISO-8601, allocators, file status, Foundation Models, HealthKit, Journaling Suggestions |
| [swiftui-uikit-and-documents.md](references/swiftui-uikit-and-documents.md) | SwiftUI migrations, text, navigation, App Intents, WebPage, UIKit scenes and windows, documents, Writing Tools, MetricKit |
| [graphics-media-and-spatial.md](references/graphics-media-and-spatial.md) | Metal, Charts, RealityKit, USDKit, SensorKit, Nearby Interaction, Broadcast Extensions |
| [cpp-and-language-interop.md](references/cpp-and-language-interop.md) | libc++, C++ ABI, safe wrappers, Swift/C++ annotations, spans, Objective-C race diagnostics |

## Apply the patch safely

1. Determine the Xcode build number and the SDK that compiled the binary.
2. Separate SDK-linked changes from runtime-only changes and beta defects.
3. Check deployment targets and test on the oldest supported device OS.
4. Treat beta workarounds as temporary and revalidate them after each seed.
5. For ABI-sensitive C++ code, rebuild all binary boundaries with one toolchain.
6. For distribution rules, distinguish upload requirements from runtime support.

## Breaking changes and required migrations

### Adopt scenes for iOS 27 SDK builds

Apps built with the iOS 27 SDK must use the scene-based lifecycle or they do not
launch. Add scene configuration and move lifecycle assumptions away from the
application delegate before changing the SDK.

### Replace deprecated document protocols

Prefer SwiftUI's unified `Document` protocol for ordinary read-write documents.
`FileDocument` and `ReferenceFileDocument` are deprecated. Update
`FileWrapperDocumentWriter` closures to accept both the new configuration and
`previous: FileWrapper?` arguments.

### Move downloadable content off On Demand Resources

On Demand Resources and `NSBundleResourceRequest` are deprecated. Use Background
Assets for downloadable app content and redesign request, storage, and update flows
instead of wrapping the old API.

### Migrate Core Data synchronization

The iOS 26 SDK rejects the old Core Data ubiquity store options. Removing those
options keeps a local store but stops synchronization. Move synchronization to
`NSPersistentCloudKitContainer` or SwiftData.

### Respect Core Data concurrency annotations

`NSManagedObject` is nonisolated and non-`Sendable`; keep it inside its context's
scope. `NSManagedObjectContext` is `Sendable`, and its perform APIs take `Sendable`
closures. Use `-com.apple.CoreData.ConcurrencyDebug 1` to expose violations.

### Upgrade transport security

Apps linked on or after iOS 26 default URLSession and Network framework connections
to TLS 1.2 minimum. The iOS 27 managed-service processes also require ATS-compliant
TLS 1.2 servers. Fix servers first; use explicit protocol configuration only for a
deliberate legacy exception.

### Remove obsolete VPN algorithms

IKEv2 no longer supports DES, 3DES, SHA1-96, SHA1-160, or Diffie-Hellman groups
below 14. Update profiles and VPN servers together.

### Replace the legacy Push to Talk entitlement

Apps built with the iOS 26 SDK cannot use
`com.apple.developer.pushkit.unrestricted-voip.ptt`. Adopt the Push to Talk
framework.

### Replace old MetricKit adoption

Do not newly adopt `MXMetricManager`, `MXMetricManagerSubscriber`,
`MXMetricPayload`, or `MXDiagnosticPayload`. Use `MetricManager`.

### Fix localized text construction

Do not interpolate nonlocalized values directly into `LocalizedStringResource`,
`String(localized:)`, or `AttributedString(localized:)`. Supply a localized value,
or use `String(describing:)` when a description is intentionally not localized.
Replace concatenated SwiftUI `Text` with interpolation so translations can reorder
content.

### Audit UIKit scene and screen access

`UIScreen.mainScreen` is deprecated. In iOS 27 SDK builds, deprecated application
status-bar accessors can return null or NaN; read status-bar state from
`UIWindowScene.statusBarManager`. Avoid assuming that an initial mirrored scene is
attached to the main screen.

### Audit C++ ABI and ordered lookup assumptions

Rebuild ABI boundaries that contain affected unordered containers or `deque` after
moving to Xcode 26. With the Xcode 27 libc++, `multimap::find` and `multiset::find`
need not return the first equivalent element; use `lower_bound` or `equal_range`
when that ordering matters.

## High-use API changes

### URLSession loading and AsyncImage

Set `URLSessionConfiguration.usesClassicLoadingMode = false` to exercise the newer
HTTP loading implementation. `AsyncImage` uses standard HTTP caching and can accept
a `URLRequest`; use `asyncImageURLSession(_:)` to supply the session to descendants.

### StoreKit entitlements and offers

Use `Transaction.currentEntitlements(for:)` rather than the singular deprecated
API so family-shared transactions are included. Introductory-offer eligibility is
false with no signed-in App Store account. The compact-JWS purchase option can
explicitly allow or block an introductory offer.

### SwiftUI navigation, gestures, and layout

Move `containerValue(_:_:)` outside a `NavigationLink` when the value must escape
its label or button style. Use `highPriorityGesture(_:isEnabled:)` to precede a
native recognizer and `simultaneousGesture(_:isEnabled:)` for equal priority.
Button-like pickers now fit by default; apply `buttonSizing(_:)` when they should
fill available space.

### SwiftUI text and menus

SDK-linked text controls infer base writing direction per paragraph. Set
`AttributedString.writingDirection` or use
`.writingDirection(strategy: .layoutBased)` when inference is inappropriate.
On the iPadOS 27 menu bar, force `.labelStyle(.titleAndIcon)` when a symbol must
remain visible.

### Documents and export

The collection overload of
`fileExporter(isPresented:documents:contentTypes:onCompletion:onCancellation:)`
exports multiple `WritableDocument` values in one dialog and returns their URLs.
Package writers can reuse the prior file wrapper to avoid rewriting unchanged
children.

### WebPage navigation

Treat `WebPage.load` results as an `AsyncSequence` of navigation events. Replace
removed `currentNavigationEvent` access with the indefinite `navigations` sequence.
URLs can be loaded directly, and HTML-string loading has a default `baseURL`.

### App Intents schemas

Use `AttributedString` for the name in `notes.createNote` and `notes.updateNote`.
Rename `calendar.deleteEvents` adoption to singular `calendar.deleteEvent`. Supply
explicit defaults for Set-valued parameters because schema defaults may not apply.

### Passkey registration

`preferImmediatelyAvailableCredentials` also controls passkey registration. It
presents registration UI only when the device can create a passkey immediately;
otherwise it presents nothing.

### Swift System file status

Use `Stat` with `FilePath`, `FileDescriptor`, or a C string, or call the new
`FilePath.stat()` and `FileDescriptor.stat()` instance methods for stat-family
operations.

## Beta and diagnostic watchpoints

- Parallel test processes can substantially delay streamed standard output.
- Rebuild Address Sanitizer tests with Xcode 26.5 or later for 27.0 systems.
- Declare all four iPad orientations to preserve continuous resizing in the beta.
- Critical alerts are automatically enabled for apps requesting notification
  permission in the beta; users must disable them in Settings if unwanted.
- `UISceneClosureConfirmation` may fail to display its dialog.
- SensorKit PPG fetches can return no samples.
- Swift UTF-16 byte-length APIs can report the wrong length, including after
  bridging to `NSString`.
- Duration or `LPLinkMetadata` intent parameters can prevent Shortcuts' described
  edit flow from opening.
- Alternate web-distributed app installation fails on distribution-ineligible
  devices even when the browser itself is under development.

## Distribution checkpoint

App Store Connect uploads must be built with Xcode 26 or later and a version 26
platform SDK.
