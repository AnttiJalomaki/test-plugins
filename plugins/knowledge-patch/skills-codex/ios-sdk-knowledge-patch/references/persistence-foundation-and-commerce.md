# Persistence, Foundation, and Commerce

## Keep Core Data values within their concurrency domain

The iOS 26 SDK imports `NSManagedObject` as nonisolated and non-`Sendable`,
`NSManagedObjectContext` as nonisolated and `Sendable`, and the `perform` and
`performBlock` family with `Sendable` closures. A rebuild can therefore expose
new concurrency warnings even when the source has not changed. (26.0)

Keep each managed object within the scope of its context. During migration,
launch with the following argument to catch violations: (26.0)

```text
-com.apple.CoreData.ConcurrencyDebug 1
```

## Remove Core Data ubiquity store options

Apps built with the iOS or macOS 26 SDK receive errors for these legacy
options: (26.0)

- `NSPersistentStoreUbiquitousContentNameKey`
- `NSPersistentStoreUbiquitousContentURLKey`
- `NSPersistentStoreUbiquitousPeerTokenOption`
- `NSPersistentStoreRemoveUbiquitousMetadataOption`
- `NSPersistentStoreUbiquitousContainerIdentifierKey`
- `NSPersistentStoreRebuildFromUbiquitousContentOption`

Older builds log warnings instead. Removing the options preserves the local
store but stops synchronization through that mechanism. Migrate synchronization
to `NSPersistentCloudKitContainer` or SwiftData.

## Parse ISO-8601 fractional seconds consistently

`ISO8601FormatStyle` accepts fractional seconds regardless of its
`includingFractionalSeconds` setting. Do not use that setting as an input
validation barrier; validate the source text separately if a protocol forbids
fractions. (26.0)

## Update the intended AdAttributionKit conversion

AdAttributionKit supports multiple simultaneous re-engagement conversions.
Read the conversion tag from the re-engagement URL parameter and pass it to
`updateConversionValue` so the intended conversion is updated. (18.4)

For development postback testing, an Xcode-built advertised app can create and
interact with development postbacks under **Settings > Developer > Ad
Attribution Testing**. This flow does not require a publisher app or prior
store distribution. (18.4)

## Apply Advanced Commerce introductory-offer policy

StoreKit supports Advanced Commerce API purchases and adds the purchase option
`introductoryOfferEligibility(compactJWS:)`. The server-signed compact JWS can
request that an introductory offer be applied even when the customer would
otherwise be ineligible, or can block redemption. Generate and evaluate that
policy as a signed server decision. (18.4)

## Consume the expanded transaction model

New transaction metadata across `AppTransaction`, `Transaction`,
`Transaction.Offer`, and `Product.SubscriptionInfo.RenewalInfo` includes
`appTransactionID`, `originalPlatform`, and `period`. The platform type for
`originalPlatform` moved to `AppStore.Platform`; its `watchOS` case was removed
and combined with `iOS`. Update exhaustive switches and persisted mappings.
(18.4)

## Preserve complete entitlement results

`Transaction.currentEntitlement(for:)` is deprecated. Use
`Transaction.currentEntitlements(for:)`, which does not omit family-shared
transactions. Because the replacement is plural, consume the complete result
rather than selecting an arbitrary first value. (18.4)

`isEligibleForIntroOffer(for:)` returns `false` when no App Store account is
signed in. Treat sign-in state separately from a definitive signed-in
ineligibility result. (18.4)
