# Networking, Security, and Distribution

## Exercise the new HTTP loading mode

The newer URL loading implementation is opt-in in iOS 18.4 and is planned to
become the default in a future release. Set `usesClassicLoadingMode` to `false`
on the configuration, then test redirects, authentication, caching, proxies,
metrics, and protocol integrations before broad rollout. (18.4)

```swift
let configuration = URLSessionConfiguration.default
configuration.usesClassicLoadingMode = false
let session = URLSession(configuration: configuration)
```

## Raise legacy endpoints to TLS 1.2

For apps linked on or after iOS 26 or macOS 26, `URLSession` and Network
framework connections default to a minimum of TLS 1.2 rather than TLS 1.0.
Upgrade services first. If a temporary exception is unavoidable, set the
minimum explicitly with
`URLSessionConfiguration.tlsMinimumSupportedProtocolVersion` or
`sec_protocol_options_set_min_tls_protocol_version`. (26.0)

## Replace removed IKEv2 cryptography

IKEv2 VPNs no longer support DES, 3DES, SHA1-96, SHA1-160, or Diffie-Hellman
groups below 14. Update both the client profile and the VPN server proposal.
(26.0)

## Respect Team ID isolation for named semaphores

For processes signed with a Team ID entitlement, `sem_open` and `sem_unlink`
cannot observe POSIX named semaphores created by a different development team.
Use an IPC mechanism designed for cross-team communication instead of assuming
the missing name is a startup race. (26.0)

## Recover affected enterprise applications

iOS and iPadOS 18.5 fix an iOS 18-era failure that could prevent some
enterprise apps from launching. A device that already encountered the issue
must have all enterprise apps uninstalled and reinstalled; installing only the
OS fix does not repair the affected installations. Plan for local application
data before removal. (18.5)

## Meet the App Store SDK upload floor

Since April 28, 2026, uploads to App Store Connect must be built with Xcode 26
or later and use an SDK for iOS 26, iPadOS 26, tvOS 26, visionOS 26, or watchOS
26. Update archive and CI images as well as local development machines.
(`app-store-sdk-requirements`)
