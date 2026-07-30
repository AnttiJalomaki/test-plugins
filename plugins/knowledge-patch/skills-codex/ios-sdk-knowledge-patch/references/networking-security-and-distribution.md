# Networking, Security, and Distribution

## HTTP loading and TLS

### New URLSession loading mode

Opt into the newer HTTP loading implementation by setting
`usesClassicLoadingMode` to `false`; the new implementation is planned to
become the default in a future release (18.4):

```swift
let configuration = URLSessionConfiguration.default
configuration.usesClassicLoadingMode = false
let session = URLSession(configuration: configuration)
```

### TLS 1.2 linked-on minimum

For apps linked on or after iOS 26 or macOS 26, URLSession and Network
framework connections default to TLS 1.2 rather than TLS 1.0. Prefer upgrading
the endpoint. A deliberate legacy exception can set
`URLSessionConfiguration.tlsMinimumSupportedProtocolVersion` or call
`sec_protocol_options_set_min_tls_protocol_version` (26.0).

### Managed system services

On 27.0 operating systems, the processes handling MDM, DDM, Automated Device
Enrollment, configuration-profile installation, app installation, and software
updates require TLS 1.2 or later with ATS-compliant cipher suites and
certificates. This is a server requirement, not just an app transport setting
(27.0-beta4).

## VPN, IPC, and entitlements

- IKEv2 no longer supports DES, 3DES, SHA1-96, SHA1-160, or Diffie-Hellman
  groups below 14. Upgrade both VPN profiles and the server (26.0).
- For processes signed with a Team ID entitlement, `sem_open` and `sem_unlink`
  cannot observe named semaphores created by another development team. Do not
  use POSIX semaphore names as cross-team IPC (26.0).
- Apps built with the iOS 26 SDK or later cannot use
  `com.apple.developer.pushkit.unrestricted-voip.ptt`. Migrate to the Push to
  Talk framework introduced in iOS 16 (26.0).

## AdAttributionKit

Multiple re-engagement conversions can coexist. Read the conversion tag from
the re-engagement URL parameter and pass it to `updateConversionValue` so the
intended conversion is updated. An advertised app built from Xcode can create
and interact with development postbacks in **Settings > Developer > Ad
Attribution Testing** without a publisher app or prior store distribution
(18.4).

Postbacks include a country code when crowd-anonymity thresholds are met
(26.6-rc).

## Authentication Services

`ASAuthorizationControllerRequestOptions.preferImmediatelyAvailableCredentials`
applies to passkey registration as well as credential requests. Registration UI
appears only when the device can create a passkey immediately; otherwise no UI
appears (26.6-rc).

## Enterprise and alternate distribution

- iOS and iPadOS 18.5 fix an iOS 18-era launch failure affecting some
  enterprise apps. A device that already encountered the issue requires all
  enterprise apps to be uninstalled and reinstalled (18.5).
- An in-development browser running on a device that is ineligible for
  alternate app distribution cannot install web-distributed apps. Installation
  starts and then fails before completion (26.6-rc).

## App Store upload requirement

Since April 28, 2026, App Store Connect uploads must be built with Xcode 26 or
later and an SDK for iOS 26, iPadOS 26, tvOS 26, visionOS 26, or watchOS 26
(app-store-sdk-requirements).
