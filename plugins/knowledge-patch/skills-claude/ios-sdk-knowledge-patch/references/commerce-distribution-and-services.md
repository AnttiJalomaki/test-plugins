# Commerce, Distribution, and Platform Services

## StoreKit purchases and entitlements

StoreKit supports Advanced Commerce API purchases and adds the purchase option `introductoryOfferEligibility(compactJWS:)`. The server-signed compact JWS can request an introductory offer for an otherwise ineligible customer or block redemption (18.4).

New metadata includes `appTransactionID`, `originalPlatform`, and `period` across `AppTransaction`, `Transaction`, `Transaction.Offer`, and `Product.SubscriptionInfo.RenewalInfo`. The type used by `originalPlatform` moved to `AppStore.Platform`; its `watchOS` case was removed and combined with `iOS` (18.4).

`Transaction.currentEntitlement(for:)` is deprecated. Use `Transaction.currentEntitlements(for:)` so family-shared transactions are not omitted. `isEligibleForIntroOffer(for:)` returns `false` when no App Store account is signed in, so require a signed-in account before interpreting that result as actual ineligibility (18.4).

## Advertising attribution

AdAttributionKit supports overlapping re-engagement conversions. Extract the conversion tag from the re-engagement URL parameter and pass it to `updateConversionValue` so the intended conversion is updated (18.4).

An advertised app built by Xcode can create and interact with development postbacks under Settings > Developer > Ad Attribution Testing without a publisher app or prior store distribution (18.4).

Postbacks include a country code when crowd-anonymity thresholds are met (26.6-rc).

## Distribution and extensions

Broadcast Extensions have a higher per-process memory limit on iOS and iPadOS 18.5, allowing higher-quality capture and streaming when resources permit (18.5).

iOS and iPadOS 18.5 resolve an iOS 18-era failure that prevented some enterprise apps from launching. An affected device still requires all enterprise apps to be uninstalled and reinstalled (18.5).

An in-development browser tested on a device that is ineligible for alternate app distribution cannot install web-distributed apps. Installation begins but fails before completion (26.6-rc).

## Nearby Interaction and live activities

An app with an active Live Activity can perform Ultra Wideband ranging through Nearby Interaction while in the background (18.4).

## Authentication services

`ASAuthorizationControllerRequestOptions.preferImmediatelyAvailableCredentials` applies to passkey registration. It presents UI only if the device can immediately create a passkey; otherwise it presents no UI (26.6-rc).

## App Intents and AssistantSchemas

The `notes.createNote` and `notes.updateNote` schemas accept `AttributedString` for their `name` parameter. `calendar.deleteEvents` is renamed to singular `calendar.deleteEvent` (27.0-beta4).

Email code adopting the `createDraft`, `updateDraft`, `replyMail`, `forwardMail`, `message`, or `draft` AssistantSchemas can fail to compile because their parameter types changed; update call sites against the new signatures (26.6-rc).

In the current beta, schema defaults might not apply to Set-valued parameters. Supply an explicit `@Parameter` default, such as an empty set. Non-SF Symbol entity images might not appear in Siri, and workout-audio entities registered through `RelevantEntities` might not appear in the Fitness media picker (27.0-beta4).

A shortcut whose app intent uses `Duration` or `LPLinkMetadata` can fail to edit through the “Describe a change” feature in the current beta (27.0-beta4).

## Health, journaling, and intelligence APIs

`HKWorkoutSession` and `HKLiveWorkoutBuilder` are available on iOS and iPadOS for workout tracking (26.6-rc).

Journaling Suggestions created on iPhone can sync securely through iCloud to adopting iPad apps. The API adds smart notifications based on routine and location, scene classifications, holiday and celebration inferences, and new pattern-based groupings (26.6-rc).

The `.contentTagging` use case accepts non-English prompts and emits tags in the prompt language. Query the exact supported set from `SystemLanguageModel(useCase: .contentTagging).supportedLanguages` (26.6-rc).

## Graphics, charts, and media limitations

`Chart3D` uses RealityKit to visualize data and mathematical surfaces in 3D on iOS 26, macOS 26, and visionOS 26 (26.6-rc).

The `realitykit_hair_surfaceshader` ShaderGraph node does not support `DiffuseLightProbeGroupComponent`; affected hair materials might not respond to diffuse light-probe-group lighting in the current beta (27.0-beta4).

SensorKit's PPG reader can return no samples in the current beta (27.0-beta4).

USDKit cannot currently read or modify some USD attribute types, and it cannot author array, vector, matrix, or quaternion values (27.0-beta4).

When using Metal 4 command encoders, add render and compute pipelines that support indirect command buffers to the residency set even though the driver does not currently enforce the requirement (26.0).

## Metrics, alerts, and content delivery

For new adoption, use `MetricManager` rather than the original `MXMetricManager`, `MXMetricManagerSubscriber`, `MXMetricPayload`, and `MXDiagnosticPayload` APIs (27.0-beta4).

On Demand Resources and `NSBundleResourceRequest` are deprecated. Move downloadable app content to Background Assets (27.0-beta4).

In the current beta, critical alerts are enabled automatically for every app that requests notification permission. A user who does not want them must disable Critical Alerts for the app in notification settings (27.0-beta4).

## Legacy Push to Talk

Apps built with the iOS 26 SDK or later cannot use `com.apple.developer.pushkit.unrestricted-voip.ptt`. Migrate to the Push to Talk framework introduced in iOS 16 (26.0).
