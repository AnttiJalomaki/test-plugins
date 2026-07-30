# Persistence, Foundation, and Commerce

## Core Data

### Concurrency imports

The iOS 26 SDK imports `NSManagedObject` as nonisolated and non-`Sendable`,
`NSManagedObjectContext` as nonisolated and `Sendable`, and the `perform` and
`performBlock` families with `Sendable` closures. Rebuilding can reveal new
warnings. Keep managed objects inside their context's scope and launch with
`-com.apple.CoreData.ConcurrencyDebug 1` while finding violations (26.0).

### Removed ubiquity store options

Apps built with the iOS or macOS 26 SDK receive errors for these options; older
builds log warnings (26.0):

- `NSPersistentStoreUbiquitousContentNameKey`
- `NSPersistentStoreUbiquitousContentURLKey`
- `NSPersistentStoreUbiquitousPeerTokenOption`
- `NSPersistentStoreRemoveUbiquitousMetadataOption`
- `NSPersistentStoreUbiquitousContainerIdentifierKey`
- `NSPersistentStoreRebuildFromUbiquitousContentOption`

Removing the options preserves the local store but stops synchronization.
Migrate synchronization to `NSPersistentCloudKitContainer` or SwiftData.

## StoreKit

### Purchases and introductory offers

StoreKit supports Advanced Commerce API purchases. The purchase option
`introductoryOfferEligibility(compactJWS:)` accepts a server-signed compact JWS
that can request an introductory offer for an otherwise-ineligible customer or
block redemption (18.4).

`isEligibleForIntroOffer(for:)` returns `false` when no App Store account is
signed in, so eligibility checks require an authenticated account (18.4).

### Transactions and platform metadata

New metadata includes `appTransactionID`, `originalPlatform`, and `period`
across `AppTransaction`, `Transaction`, `Transaction.Offer`, and
`Product.SubscriptionInfo.RenewalInfo`. The platform type for
`originalPlatform` moved to `AppStore.Platform`; its `watchOS` case was removed
and folded into `iOS` (18.4).

`Transaction.currentEntitlement(for:)` is deprecated. Use
`Transaction.currentEntitlements(for:)` so family-shared transactions are not
omitted (18.4).

## Localization, formatting, and strings

### Localized interpolation

Interpolating a nonlocalized type into `LocalizedStringResource`,
`String(localized:)`, or `AttributedString(localized:)` produces a deprecation
warning. Supply a localized value or explicitly wrap an intentional description
in `String(describing:)`. SwiftUI `Text` concatenation with `+` is also
deprecated; use interpolation so localization can reorder the content (26.0).

### ISO-8601 parsing

- `ISO8601FormatStyle` permits fractional seconds regardless of the
  `includingFractionalSeconds` setting (26.0).
- It also accepts time-zone offsets containing only hours (26.6-rc).

### UTF-16 byte lengths

`lengthOfBytes(using: .utf16)` and Objective-C
`lengthOfBytesUsingEncoding:` with `NSUTF16StringEncoding` or
`NSUnicodeStringEncoding` can return the wrong value for Swift strings,
including strings bridged to `NSString`. Avoid relying on the result until the
known issue is resolved (26.6-rc).

## System and XML APIs

### libxml2 allocation

The libxml2 custom allocation API is deprecated. Replace `xmlMalloc()` and
`xmlMallocAtomic()` with `malloc()`, `xmlRealloc()` with `realloc()`,
`xmlFree()` with `free()`, and `xmlMemStrdup()` with `strdup()`. Stop setting
allocators with `xmlMemSetup()`, `xmlMemGet()`, `xmlGcMemSetup()`,
`xmlGcMemGet()`, or their global variables; libxml2 and libxslt use the system
allocator internally (18.4).

### File ports and file status

`fileport_makeport(2)` and `fileport_makefd(2)` are public APIs with manual
pages (18.4).

Swift System adds `Stat` initializers for a `FilePath`, `FileDescriptor`, or C
string, plus `FilePath.stat()` and `FileDescriptor.stat()` instance methods for
the `stat`, `lstat`, `fstat`, and `fstatat` family (27.0-beta4).

## Intelligence, health, and journaling

The Foundation Models `.contentTagging` use case accepts non-English prompts
and produces tags in the prompt language. Query the exact supported set with
`SystemLanguageModel(useCase: .contentTagging).supportedLanguages` (26.6-rc).

`HKWorkoutSession` and `HKLiveWorkoutBuilder` are available on iOS and iPadOS
for workout tracking (26.6-rc).

Journaling Suggestions created on iPhone can securely sync through iCloud to
iPad apps adopting the API. The API adds routine- and location-based smart
notifications, scene classifications, holiday and celebration inferences, and
new pattern-based groupings (26.6-rc).
