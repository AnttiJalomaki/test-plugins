---
name: ios-sdk-knowledge-patch
description: iOS SDK
version: iOS 26 / Xcode 26
license: MIT
metadata:
  author: Nevaberry
---



# iOS SDK Knowledge Patch

Use this skill when maintaining Apple-platform projects that depend on recent
iOS SDK, Xcode, SwiftUI, UIKit, Foundation, StoreKit, Core Data, networking,
Metal, or C-family toolchain behavior.

Check the project's Xcode version, deployment targets, linked SDK, enabled
entitlements, and affected frameworks before applying guidance. Several
behaviors are SDK-linked: the runtime OS alone does not determine them.

## Reference index

| Reference | Topics |
| --- | --- |
| [C++, Swift interop, and runtime](references/cpp-and-language-interop.md) | C and POSIX APIs, libxml2, libc++, Swift/C++ interop, Objective-C races |
| [Graphics, media, and spatial APIs](references/graphics-media-and-spatial.md) | Broadcast Extensions, Nearby Interaction, Push to Talk, Metal 4 |
| [Networking, security, and distribution](references/networking-security-and-distribution.md) | URL loading, TLS, IKEv2, semaphores, enterprise and App Store distribution |
| [Persistence, Foundation, and commerce](references/persistence-foundation-and-commerce.md) | Core Data, Foundation formatting, AdAttributionKit, StoreKit |
| [SwiftUI, UIKit, and documents](references/swiftui-uikit-and-documents.md) | Localization, controls, navigation, gestures, text, Writing Tools, screen access |
| [Toolchain, build, and testing](references/toolchain-build-and-testing.md) | Xcode requirements, devices, Simulator, linker, modules, packages, tests |

## Start with migration blockers

Before changing code, identify whether the project is merely running on a new
OS or is also being rebuilt with a new SDK. Prioritize these build- and
link-sensitive changes:

- Remove Core Data ubiquity options rejected by the iOS 26 SDK. Preserve the
  local store, then move synchronization to `NSPersistentCloudKitContainer`
  or SwiftData.
- Replace the unrestricted VoIP Push to Talk entitlement with the Push to Talk
  framework for apps built with the iOS 26 SDK or later.
- Upgrade IKEv2 profiles and servers away from DES, 3DES, SHA1-96, SHA1-160,
  and Diffie-Hellman groups below 14.
- Replace `Transaction.currentEntitlement(for:)` with
  `Transaction.currentEntitlements(for:)` so family-shared transactions are
  included.
- Replace libxml2 custom allocation entry points and allocator configuration
  with the system `malloc`, `realloc`, `free`, and `strdup` family.
- Treat `UIScreen.mainScreen` as deprecated on iOS, tvOS, and visionOS 26.
- Audit binary boundaries containing libc++ unordered containers or `deque`
  when empty allocators, comparators, or hashers participate in layout.

## Verify the toolchain and delivery path

Use the Xcode release that supplies the intended SDK and runs on the build
host. Do not infer host compatibility from an application's deployment target.
See the toolchain reference for exact host and device requirements.

For App Store submissions, verify the current upload floor before archiving.
The documented requirement in this patch applies from April 28, 2026: use
Xcode 26 or later and a version 26 platform SDK.

When diagnosing build or test failures:

1. Record the Xcode, SDK, Swift language, and deployment-target versions.
2. Distinguish a compiler or linker failure from a Simulator-runtime defect.
3. Check whether Swift explicit modules became active automatically.
4. Retest extensions and device-only features on physical hardware where the
   Simulator does not expose them.
5. Capture diagnostics with `devicectl diagnose`; it now collects from the Mac
   and every available device by default.

## Core Data migration checks

The iOS 26 SDK changes concurrency imports:

- `NSManagedObject` is nonisolated and non-`Sendable`.
- `NSManagedObjectContext` is nonisolated and `Sendable`.
- `perform` and `performBlock` closures are imported as `Sendable`.

Keep managed objects inside their context's scope. During migration, run with
`-com.apple.CoreData.ConcurrencyDebug 1` to turn hidden misuse into actionable
failures.

Remove all obsolete `NSPersistentStoreUbiquitous*` keys and the related rebuild
and metadata options. Removing them keeps the local store but stops that legacy
sync path, so plan the replacement before shipping.

## Networking and security defaults

Apps linked on or after iOS 26 default `URLSession` and Network framework
connections to TLS 1.2 minimum. Prefer upgrading endpoints. If a legacy service
must be reached temporarily, set the minimum explicitly with
`URLSessionConfiguration.tlsMinimumSupportedProtocolVersion` or
`sec_protocol_options_set_min_tls_protocol_version`.

The newer URL loading stack is opt-in in iOS 18.4:

```swift
let configuration = URLSessionConfiguration.default
configuration.usesClassicLoadingMode = false
let session = URLSession(configuration: configuration)
```

Team-signed processes cannot use POSIX named semaphores to observe names
created by a different development team. Treat this as an isolation boundary,
not an intermittent synchronization failure.

## SwiftUI and UIKit migration checks

For SDK-linked UI changes, test both newly built binaries and older binaries on
the same OS. In particular:

- Paragraph direction for `Text`, `TextEditor`, and `TextField` follows content
  in new builds. Set `AttributedString.writingDirection` per paragraph, or use
  `.writingDirection(strategy: .layoutBased)` when layout direction must win.
- Button-like picker styles use fitted sizing. Apply `buttonSizing(_:)` when
  the picker should fill available space.
- `NavigationLink` is a single view in list contexts. Put
  `containerValue(_:_:)` outside the link when the value must escape its label
  or `ButtonStyle`.
- Use `highPriorityGesture(_:isEnabled:)` to precede native recognizers and
  `simultaneousGesture(_:isEnabled:)` for equal priority.
- Use `includesTextListMarkers` to choose whether TextKit 2 and Writing Tools
  attributed strings contain list-marker text.

Prefer `Text` interpolation over concatenation so localization can reorder
content. When interpolating a deliberately nonlocalized value into a localized
string, wrap it with `String(describing:)`; otherwise provide a localized value.

## StoreKit and attribution checks

Advanced Commerce purchases and compact-JWS introductory-offer controls are
available through StoreKit. Treat the signed eligibility option as a deliberate
server decision: it can either apply an offer to an otherwise ineligible
customer or prevent redemption.

When consuming transaction data, account for `appTransactionID`,
`originalPlatform`, and `period`. `originalPlatform` uses `AppStore.Platform`;
the former `watchOS` case is represented by `iOS`.

Do not present an unsigned user as introductory-offer eligible:
`isEligibleForIntroOffer(for:)` returns `false` when no App Store account is
signed in.

For AdAttributionKit re-engagement, read the conversion tag from the
re-engagement URL and pass it to `updateConversionValue`. Development postbacks
for Xcode-built advertised apps can be exercised from the device's Ad
Attribution Testing settings without a publisher app or prior store release.

## C, C++, Swift, and Objective-C checks

Define `BUILD_FOR_APPLE_SDK` before including hvf headers when availability
checking is required:

```c
#define BUILD_FOR_APPLE_SDK 1
```

Do not build new code around the restored generic `std::char_traits` template.
Its return in Xcode 16.4 is temporary compatibility for nonstandard
instantiations, and the base template remains deprecated.

`MutableSpan` may be passed as an `inout` parameter without an experimental
feature. Swift also inherits `SWIFT_SHARED_REFERENCE` from an annotated C++
base type.

A crash that reads `0x400000000000bad0` from a synthesized Objective-C
nonatomic property setter is a race signal. Fix concurrent property access;
do not treat the sentinel as application data.

## Media and graphics checks

An active Live Activity permits background Ultra Wideband ranging through
Nearby Interaction. Broadcast Extensions have a higher memory limit on iOS and
iPadOS 18.5, but quality still depends on available system resources.

With Metal 4, place render and compute pipelines that support indirect command
buffers in the residency set. The current driver may not enforce the rule, but
code should satisfy it now.

## Final validation

Before declaring a migration complete:

- Build every supported target with the selected Xcode release.
- Exercise network connections against their real TLS and VPN endpoints.
- Run Core Data with concurrency debugging enabled.
- Test StoreKit account-signed-out and family-sharing paths.
- Verify bidirectional text, picker sizing, list markers, and gesture ordering.
- Check binary compatibility across C++ framework boundaries.
- Run device-only extensions on hardware.
- Confirm archive tooling meets the App Store upload requirement.
