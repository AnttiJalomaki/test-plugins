---
name: ios-sdk-knowledge-patch
description: iOS SDK
version: iOS 27 beta 4 / Xcode 27 beta 4
license: MIT
metadata:
  author: Nevaberry
---


# iOS SDK Knowledge Patch

Use this skill when an iOS, iPadOS, or closely related Apple-platform task depends on recent SDK-linked behavior, Xcode toolchain changes, renamed or deprecated APIs, beta limitations, or current upload requirements.

## Reference index

| Reference | Topics |
| --- | --- |
| [toolchain-build-and-testing.md](references/toolchain-build-and-testing.md) | Xcode hosts and SDKs, device floors, build settings, Simulator, diagnostics, testing, Interface Builder, package builds, uploads |
| [language-runtime-and-interop.md](references/language-runtime-and-interop.md) | Swift concurrency and localization, Objective-C races, C and C++ compatibility, libc++, libxml2, Swift System |
| [networking-data-and-security.md](references/networking-data-and-security.md) | URL loading, TLS, VPN, Core Data, ISO-8601, semaphores, WebPage, UTF-16 |
| [ui-text-scenes-and-documents.md](references/ui-text-scenes-and-documents.md) | SwiftUI, UIKit, TextKit, Writing Tools, navigation, menus, scenes, external displays, documents |
| [commerce-distribution-and-services.md](references/commerce-distribution-and-services.md) | StoreKit, attribution, enterprise and web distribution, passkeys, intents, health, charts, diagnostics, content delivery |

## Apply the patch

1. Determine the Xcode version, linked SDK, deployment targets, and affected operating systems.
2. Separate compile-time toolchain changes from behavior gated by the SDK used to build the app.
3. Apply only guidance whose version is at or below the project's selected toolchain or SDK.
4. Treat beta limitations as temporary and recheck them when moving to a newer seed.
5. Prefer the project's manifests, build settings, code, tests, and observed behavior when they conflict with general guidance.

## Breaking changes and required migrations

### Adopt scenes for iOS 27 SDK builds

Apps built with the iOS 27 SDK must use the scene-based lifecycle. A build that still relies solely on the application-delegate lifecycle can fail to launch.

Audit the app manifest, scene configuration, app and scene delegates, URL handling, restoration, and notification routing before changing the SDK.

### Remove obsolete Core Data ubiquity options

The iOS 26 SDK rejects the old `NSPersistentStoreUbiquitous*` options and `NSPersistentStoreRemoveUbiquitousMetadataOption`. Remove them and migrate synchronization to `NSPersistentCloudKitContainer` or SwiftData.

Removing the options preserves the local store, but it stops that legacy synchronization path. Plan migration rather than treating the compiler error as a harmless rename.

### Replace removed or changed communication behavior

- Apps built with the iOS 26 SDK cannot use `com.apple.developer.pushkit.unrestricted-voip.ptt`; use the Push to Talk framework.
- IKEv2 no longer accepts DES, 3DES, SHA1-96, SHA1-160, or Diffie-Hellman groups below 14. Update both profiles and servers.
- Linked-on-or-after OS 26 networking raises the default minimum to TLS 1.2.
- Managed services on OS 27 also require TLS 1.2 with ATS-compliant cipher suites and certificates.

### Migrate deprecated downloadable content

On Demand Resources and `NSBundleResourceRequest` are deprecated. Move remotely downloadable app content to Background Assets.

### Update SwiftUI document code

Use the combined `Document` protocol for ordinary read-write documents. `ReferenceFileDocument` and `FileDocument` are deprecated.

Update `FileWrapperDocumentWriter.makeFileWrapper` closures to accept the second `previous: FileWrapper?` parameter. Package writers can mutate and return the previous wrapper to avoid rewriting unchanged children.

### Update WebPage navigation handling

`WebPage.load` returns an `AsyncSequence` of navigation events. Replace removed `currentNavigationEvent` usage with the indefinite `navigations` sequence, and ensure consumers have explicit cancellation or lifetime management.

### Correct App Intent schema adoption

- `calendar.deleteEvents` is now `calendar.deleteEvent`.
- `notes.createNote` and `notes.updateNote` accept `AttributedString` for `name`.
- Several email schemas changed parameter types and can require source updates.
- Explicitly provide defaults for Set-valued schema parameters because schema defaults may not apply in the current beta.

### Repair C++ assumptions

- Do not assume `multimap::find` or `multiset::find` returns the first equivalent element; use `lower_bound` or `equal_range`.
- Fix comparators that are not strict weak orders. `_LIBCPP_ENABLE_LEGACY_TREE_LOWER_UPPER_BOUND` is only a temporary compatibility switch.
- Revalidate ABI when standard containers share empty allocator, comparator, or hasher types with enclosing layouts.
- The restored generic `std::char_traits` base template is temporary compatibility, not a reason to retain nonstandard instantiations.

## High-value SDK-linked behavior

### Networking

Opt into the newer HTTP loading path before it becomes the default:

```swift
let configuration = URLSessionConfiguration.default
configuration.usesClassicLoadingMode = false
let session = URLSession(configuration: configuration)
```

For `AsyncImage`, standard HTTP caching is automatic in the iOS 27 SDK. Use a `URLRequest` initializer to select a per-image cache policy, and `asyncImageURLSession(_:)` to provide the session for descendant images.

### StoreKit and attribution

- Prefer `Transaction.currentEntitlements(for:)` over deprecated `currentEntitlement(for:)` so family-shared transactions are included.
- Treat `isEligibleForIntroOffer(for:) == false` as inconclusive when no App Store account is signed in.
- Advanced Commerce purchases and `introductoryOfferEligibility(compactJWS:)` support server-signed introductory-offer control.
- For overlapping AdAttributionKit re-engagement conversions, pass the conversion tag from the re-engagement URL to `updateConversionValue`.

### SwiftUI interaction and layout

- Use `highPriorityGesture(_:isEnabled:)` to precede an existing native recognizer and `simultaneousGesture(_:isEnabled:)` for equal priority.
- Move `containerValue(_:_:)` outside a `NavigationLink` when a value must escape the label or its button style.
- Apply `buttonSizing(_:)` when a button-style picker must expand rather than use fitted sizing.
- Use text interpolation instead of concatenating `Text` values so localization can reorder content.
- Use `toolbarMinimizationBehavior`, replacing `toolbarMinimizeBehavior`.
- Use `.bordered` instead of the soft-deprecated `.squareBorder` and `.roundedBorder` text-field styles.

### Text direction and selection

SDK-linked text controls infer base direction per paragraph. Set `AttributedString.writingDirection` for explicit paragraph direction or `.writingDirection(strategy: .layoutBased)` to retain layout-based behavior.

With the iOS or iPadOS 27 SDK, `.textSelection(.enabled)` on `Text` activates system interactive selection.

### Core Data concurrency

The iOS 26 imports make managed objects non-`Sendable`, contexts `Sendable`, and context execution closures `Sendable`. Keep each managed object inside its context's scope.

Use this launch argument while finding violations:

```text
-com.apple.CoreData.ConcurrencyDebug 1
```

### Passkey registration

`ASAuthorizationControllerRequestOptions.preferImmediatelyAvailableCredentials` also governs registration. It shows UI only when the device can immediately create a passkey; otherwise it shows no UI.

## Deprecation map

| Old API or practice | Replacement or action |
| --- | --- |
| `Transaction.currentEntitlement(for:)` | `Transaction.currentEntitlements(for:)` |
| Core Data ubiquity store options | `NSPersistentCloudKitContainer` or SwiftData |
| PushKit unrestricted VoIP PTT entitlement | Push to Talk framework |
| `UIScreen.mainScreen` | Obtain the screen from the relevant window or scene |
| Deprecated `UIApplication` status-bar accessors | `UIWindowScene.statusBarManager` |
| `Text` concatenation with `+` | `Text` interpolation |
| `.squareBorder` / `.roundedBorder` | `.bordered` |
| `toolbarMinimizeBehavior` | `toolbarMinimizationBehavior` |
| `ReferenceFileDocument` / `FileDocument` | `Document` |
| On Demand Resources | Background Assets |
| Original MetricKit manager and payload APIs for new adoption | `MetricManager` |
| libxml2 custom allocation APIs | System `malloc`, `realloc`, `free`, and `strdup` |

## Toolchain triage

### Swift explicit modules

Xcode 26 enables Swift explicit modules by default except for pre-Swift-5 language modes and Swift/C++ interoperability. If the change causes a severe compatibility failure, temporarily set:

```text
SWIFT_ENABLE_EXPLICIT_MODULES=NO
```

Keep the opt-out narrow and investigate the incompatible dependency or build graph.

### Interface Builder on build servers

Xcode 27 compiles UIKit Interface Builder documents in `toolchain` mode by default, removing the Simulator-runtime dependency. During migration, opt out with `IBC_COCOATOUCH_COMPILER_MODE=simulator` or:

```sh
ibtool --cocoatouch-compiler-mode simulator
```

### Universal macOS output

For macOS 27 and DriverKit 27, `ARCHS_STANDARD` omits `x86_64`. Add it explicitly to `ARCHS` if a Universal binary is still required. Also account for the macOS deployment-target floor of 11.0.

### Attachments and exit tests

Swift Testing supports tests for code that terminates via `precondition()`, `fatalError()`, or another process exit. Choose attachment output with:

```sh
swift test --attachments-path <directory>
```

## Known-issue checks

- Parallel tests under Xcode 27 can significantly delay multi-process `stdout` and `stderr` streaming.
- Address Sanitizer builds from Xcode 26.4 or older may not launch on OS 27; build those tests with Xcode 26.5 or later.
- Safari extensions are absent from iOS and visionOS Simulator; test them on a device.
- `-stack_size` can fail for app-bundle targets in Xcode 16.3.
- Swift UTF-16 byte-length APIs can report an incorrect result, including through bridged `NSString`.
- In the current beta, `UISceneClosureConfirmation` can fail to present, SensorKit PPG can return no samples, and several iPad or Mirroring window behaviors regress.
- Critical alerts are automatically enabled for apps that request notification permission in the current beta; users must disable them in Settings if unwanted.

## Review checklist

- Confirm the exact Xcode build and linked SDK, not only the deployment target.
- Search for every deprecated or removed symbol named in the relevant reference.
- Inspect entitlements, `Info.plist`, scene manifests, and build settings.
- Retest networking against legacy and managed-service endpoints.
- Exercise localization with bidirectional, interpolated, and paragraph-rich text.
- Run concurrency diagnostics around Core Data and nonatomic Objective-C state.
- Validate menus, pickers, navigation links, gestures, documents, and external-display scenes on the affected OS.
- Test beta-specific UI and windowing paths on physical devices where Simulator coverage is insufficient.
- Verify distribution builds satisfy the current App Store Xcode and SDK requirement.
