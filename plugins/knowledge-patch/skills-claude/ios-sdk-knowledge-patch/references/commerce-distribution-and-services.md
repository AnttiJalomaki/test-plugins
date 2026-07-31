# Commerce, Distribution, and Platform Services

Use this reference for advertising attribution, StoreKit purchases and
entitlements, background platform capabilities, managed distribution, and
submission requirements.

## AdAttributionKit Re-engagement Conversions

On iOS 18.4 (`18.4`), AdAttributionKit supports multiple simultaneous
re-engagement conversions. Read the conversion tag from the re-engagement URL
parameter and pass it to `updateConversionValue` so the update reaches the
intended conversion rather than another active conversion.

An advertised app built by Xcode can create and interact with development
postbacks without a publisher app and without having previously been distributed
through the store. Enable and inspect them under **Settings > Developer > Ad
Attribution Testing**.

## Advanced Commerce and Introductory Offers

StoreKit in iOS 18.4 (`18.4`) adds purchase support for the Advanced Commerce
API and the `introductoryOfferEligibility(compactJWS:)` purchase option. The
server-signed compact JWS can request either of two outcomes:

- Apply an introductory offer even if StoreKit would otherwise consider the
  customer ineligible.
- Prevent introductory-offer redemption.

Treat the JWS as server-issued purchase policy, not merely a local eligibility
hint.

## StoreKit Transaction Metadata

The iOS 18.4 SDK (`18.4`) adds metadata named `appTransactionID`,
`originalPlatform`, and `period` across `AppTransaction`, `Transaction`,
`Transaction.Offer`, and `Product.SubscriptionInfo.RenewalInfo`.

The platform type used by `originalPlatform` is now `AppStore.Platform`. Its
former `watchOS` case was removed and folded into `iOS`; do not write exhaustive
logic that still expects a distinct watchOS case.

## Entitlements and Signed-In Account State

In iOS 18.4 (`18.4`), `Transaction.currentEntitlement(for:)` is deprecated.
Use `Transaction.currentEntitlements(for:)`; the singular API can omit
family-shared transactions.

`isEligibleForIntroOffer(for:)` returns `false` when no App Store account is
signed in. Establish signed-in account state before treating `false` as the
customer's actual eligibility.

## Background Nearby Interaction

An app with an active Live Activity can perform Ultra Wideband ranging through
Nearby Interaction while running in the background on iOS 18.4 (`18.4`). Tie
the background ranging experience to the Live Activity lifecycle.

## Broadcast Extension Memory

iOS and iPadOS 18.5 (`18.5`) raise the per-process memory limit for Broadcast
Extensions. Use the additional headroom for higher-quality capture and streaming
only when system resources permit; it is not an unconditional memory guarantee.

## Enterprise App Launch Recovery

iOS and iPadOS 18.5 (`18.5`) fix an iOS 18-era failure that could prevent some
enterprise apps from launching. A device that already encountered the failure
must uninstall and reinstall all enterprise apps to recover; updating in place
is insufficient.

## Push to Talk Migration

Apps built with the iOS 26 SDK (`26.0`) can no longer use the legacy
`com.apple.developer.pushkit.unrestricted-voip.ptt` entitlement. Migrate Push to
Talk behavior to the Push to Talk framework introduced in iOS 16.

## App Store SDK Requirement

The App Store submission rule tracked as `app-store-sdk-requirements` has applied
since April 28, 2026. Uploads to App Store Connect must be built with Xcode 26 or
later and use a version 26 SDK for the submitted platform:

- iOS 26
- iPadOS 26
- tvOS 26
- visionOS 26
- watchOS 26

The archive's build tool and SDK must satisfy this upload gate.
