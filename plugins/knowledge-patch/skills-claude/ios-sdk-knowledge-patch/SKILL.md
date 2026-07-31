---
name: ios-sdk-knowledge-patch
description: iOS SDK
version: iOS 26 / Xcode 26
license: MIT
metadata:
  author: Nevaberry
---




# iOS SDK Knowledge Patch

Use this skill when implementing, migrating, building, testing, or distributing
software against Apple platform SDKs. Start with the workflow below, then load
only the topic references relevant to the task.

## Working Method

1. Identify the Xcode version, SDK used to build, deployment target, host macOS,
   and affected runtime OS.
2. Separate compile-time, SDK-linked, and runtime behavior. Several migrations
   apply because of the SDK used to build even when the deployment target is
   older.
3. Classify the task with the reference index and read the matching file before
   recommending code or build-setting changes.
4. Preserve explicit compatibility branches when supporting binaries built with
   older SDKs; do not infer behavior solely from the device OS version.
5. Treat deprecations, security defaults, ABI changes, and distribution gates as
   migration work rather than optional cleanup.
6. Reproduce known toolchain failures with the exact Xcode and Simulator runtime
   before changing application code.

## Reference Index

| Reference | Topics |
| --- | --- |
| [Commerce, Distribution, and Platform Services](references/commerce-distribution-and-services.md) | Ad attribution, StoreKit, Nearby Interaction, broadcast and enterprise apps, Push to Talk, App Store submission |
| [Language, Runtime, and Interoperability](references/language-runtime-and-interop.md) | C APIs, libxml2, Objective-C races, Swift modules and spans, C++ source and ABI compatibility |
| [Networking, Data, and Security](references/networking-data-and-security.md) | URLSession, Core Data, ISO-8601, POSIX semaphores, VPN cryptography, TLS |
| [Toolchain, Build, and Testing](references/toolchain-build-and-testing.md) | Xcode requirements, device diagnostics, Simulator limits, linking, Metal, package builds, Swift Testing |
| [UI, Text, Scenes, and Documents](references/ui-text-scenes-and-documents.md) | SwiftUI, Writing Tools, localization, text direction, TextKit, navigation, gestures, screen access |

## Critical Migration Checks

### Satisfy the App Store build gate

For App Store Connect uploads, use Xcode 26 or later and a version 26 SDK for
the submitted Apple platform. A deployment target below 26 does not replace the
upload SDK requirement.

Read [Commerce, Distribution, and Platform Services](references/commerce-distribution-and-services.md)
for the affected platforms and effective date.

### Remove retired Core Data ubiquity options

Do not pass the removed ubiquity store keys when building with the new SDK.
Removing the keys leaves a local store but does not preserve synchronization;
migrate syncing to `NSPersistentCloudKitContainer` or SwiftData.

Also audit context confinement after rebuilding: managed objects remain
non-`Sendable`, while contexts and their execution closures have updated
concurrency annotations.

### Recheck transport security

Expect `URLSession` and Network framework connections from newly linked apps to
default to TLS 1.2 minimum. Configure a lower minimum only for an intentional
legacy endpoint and treat it as a temporary compatibility exception.

Remove DES, 3DES, SHA-1 variants, and Diffie-Hellman groups below 14 from both
IKEv2 profiles and servers.

### Migrate legacy Push to Talk access

The unrestricted PushKit VoIP Push to Talk entitlement is unavailable to apps
built with the new SDK. Use the Push to Talk framework instead.

### Fix localization diagnostics

Do not interpolate a nonlocalized type directly into localized-string APIs.
Provide a localized value or make intentional descriptive conversion explicit
with `String(describing:)`.

Replace SwiftUI `Text` concatenation with interpolation so translators can
reorder content.

### Replace libxml2 custom allocation

Replace libxml2 allocation entry points with the system allocator and stop
installing custom allocator callbacks. libxml2 and libxslt now allocate through
the system allocator internally.

### Audit C++ compatibility boundaries

Do not rely long-term on the restored generic `std::char_traits` base template;
nonstandard instantiations compile again only as temporary compatibility.

Revalidate persisted layouts, binary interfaces, and cross-module types that
contain affected unordered containers or `std::deque` with shared empty
allocator, comparator, or hasher bases.

### Audit global screen lookup

`UIScreen.mainScreen` is deprecated in iOS 26, tvOS 26, and visionOS 26.

## SDK-Linked UI Behavior

### Preserve paragraph direction intentionally

`Text`, `TextEditor`, and `TextField` infer base direction from each paragraph's
content in newly built apps. Apply `AttributedString.writingDirection` for
content-driven control or `.writingDirection(strategy: .layoutBased)` when the
layout direction must win.

### Recheck picker and navigation layout

Button-like pickers use fitted sizing by default; apply `buttonSizing(_:)` when
they must fill available width.

`NavigationLink` now contributes one view in list contexts. Move
`containerValue(_:_:)` outside the link when a label or `ButtonStyle` previously
relied on that value escaping the link boundary.

### Set gesture precedence explicitly

Use `highPriorityGesture(_:isEnabled:)` to precede an existing native recognizer
and `simultaneousGesture(_:isEnabled:)` for equal priority.

### Decide whether list markers belong in text

Use `includesTextListMarkers` on the relevant TextKit 2 and Writing Tools types.
Do not assume attributed-string paragraphs include their rendered list marker.

## High-Value APIs and Capabilities

### Select URLSession loading behavior

Set `usesClassicLoadingMode` to `false` on a `URLSessionConfiguration` to opt in
to the newer HTTP loading mode:

```swift
let configuration = URLSessionConfiguration.default
configuration.usesClassicLoadingMode = false
let session = URLSession(configuration: configuration)
```

Keep the opt-in explicit while classic mode remains the default.

### Update the intended advertising conversion

For simultaneous AdAttributionKit re-engagement conversions, extract the
conversion tag from the re-engagement URL and pass it to
`updateConversionValue`. Development postbacks can be exercised through the
device's Ad Attribution Testing settings for an Xcode-built advertised app.

### Handle StoreKit entitlements completely

Use `Transaction.currentEntitlements(for:)`, not the deprecated singular API,
so family-shared transactions are not lost. Require a signed-in App Store
account before interpreting introductory-offer eligibility.

For Advanced Commerce, use the signed compact JWS purchase option when the
server must explicitly allow or block introductory-offer redemption.

### Range during a Live Activity

An active Live Activity permits an app to use Nearby Interaction for Ultra
Wideband ranging while the app is in the background.

### Add contextual section actions

Use `sectionActions(content:)` for actions associated with a SwiftUI `Section`.
Account for platform presentation: trailing controls on macOS and separate form
rows on iOS and iPadOS.

### Test process termination

Use Swift Testing exit tests for code paths that call `precondition()`,
`fatalError()`, or otherwise terminate the test process. Set
`swift test --attachments-path <directory>` when attachment placement matters.

## Diagnostics and Compatibility Tactics

- Enable `-com.apple.CoreData.ConcurrencyDebug 1` while investigating managed
  object confinement failures.
- A nonatomic Objective-C property crash on `0x400000000000bad0` (`0xbad0` on
  32-bit watchOS) indicates unsafe concurrent access; fix synchronization
  rather than masking the value.
- Define `BUILD_FOR_APPLE_SDK` before importing hvf headers when availability
  checking is required.
- Opt out with `SWIFT_ENABLE_EXPLICIT_MODULES=NO` only for severe explicit-module
  compatibility failures and retain a plan to remove the workaround.
- Do not add back the old `ENABLE_DEBUG_DYLIB=NO` workaround merely because a
  target uses `LD_CLIENT_NAME`.
- Test Safari extensions on hardware because they are absent from iOS and
  visionOS Simulator.
- Treat an iOS 18.3 Simulator-wide `NSURLSession` timeout under older Xcode as a
  runtime/toolchain defect before rewriting networking code.
- Remember that `devicectl diagnose` collects from the Mac and every available
  device by default; plan collection time and artifact handling accordingly.

## Before Shipping

1. Confirm the build host and device-debugging floor for the selected Xcode.
2. Search for removed APIs, keys, entitlements, cryptography, and global screen
   assumptions.
3. Exercise SDK-linked UI behavior on each supported OS family.
4. Rebuild Swift, Objective-C, and C++ boundaries and inspect new concurrency or
   ABI diagnostics.
5. Test StoreKit, attribution, networking, and enterprise recovery with the
   account and device state each path requires.
6. Run unit, exit, Simulator, and device tests; keep toolchain limitations
   separate from product defects.
7. Verify the archive's Xcode and SDK versions before uploading.
