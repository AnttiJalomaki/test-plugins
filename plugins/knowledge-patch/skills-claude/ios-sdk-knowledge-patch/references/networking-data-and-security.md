# Networking, Data, and Security

## HTTP loading and image requests

### URLSession loading mode

Set `URLSessionConfiguration.usesClassicLoadingMode` to `false` to opt into the newer HTTP loading implementation before it becomes the default (18.4):

```swift
let configuration = URLSessionConfiguration.default
configuration.usesClassicLoadingMode = false
let session = URLSession(configuration: configuration)
```

### AsyncImage caching

`AsyncImage` automatically uses standard HTTP caching in the iOS 27 SDK. New initializers accept a `URLRequest` for per-image `cachePolicy` control, and `asyncImageURLSession(_:)` supplies the `URLSession` for descendant async images (27.0-beta4).

## Transport security

### Application networking

For apps linked on or after iOS 26 or macOS 26, `URLSession` and Network framework connections default to a minimum of TLS 1.2 rather than TLS 1.0. If a legacy endpoint must be reached during migration, set `URLSessionConfiguration.tlsMinimumSupportedProtocolVersion` or call `sec_protocol_options_set_min_tls_protocol_version` explicitly (26.0).

### Managed system services

On version 27 operating systems, system processes used for MDM, DDM, Automated Device Enrollment, configuration-profile installation, app installation, and software update require TLS 1.2 or later plus ATS-compliant cipher suites and certificates. Correct the server; these requests are not controlled by an app's URL session (27.0-beta4).

### IKEv2 VPN

IKEv2 no longer supports DES, 3DES, SHA1-96, SHA1-160, or Diffie-Hellman groups below 14. Update both the deployed VPN profiles and server configuration to stronger algorithms (26.0).

## WebPage navigation

`WebPage.load` APIs return an `AsyncSequence` of relevant navigation events. `currentNavigationEvent` is removed; consume the indefinite `navigations` sequence and arrange cancellation based on the owning task or view lifetime (26.6-rc).

`WebPage` can also load a URL directly. Its HTML-string loading API now defaults `baseURL`.

## Core Data persistence

Apps built with the iOS or macOS 26 SDK receive errors for these removed ubiquity options (26.0):

- `NSPersistentStoreUbiquitousContentNameKey`
- `NSPersistentStoreUbiquitousContentURLKey`
- `NSPersistentStoreUbiquitousPeerTokenOption`
- `NSPersistentStoreRemoveUbiquitousMetadataOption`
- `NSPersistentStoreUbiquitousContainerIdentifierKey`
- `NSPersistentStoreRebuildFromUbiquitousContentOption`

Older builds log warnings. Removing the options preserves the local store without synchronization. Migrate cloud synchronization to `NSPersistentCloudKitContainer` or SwiftData.

## Process isolation

For processes signed with a Team ID entitlement, `sem_open` and `sem_unlink` can no longer observe POSIX named semaphores created by another development team. Do not use a shared semaphore name as cross-team IPC (26.0).

## String and date parsing edge cases

- `ISO8601FormatStyle` permits fractional seconds regardless of `includingFractionalSeconds` (26.0).
- It also accepts hours-only time-zone offsets (26.6-rc).
- `lengthOfBytes(using: .utf16)` and Objective-C `lengthOfBytesUsingEncoding:` with `NSUTF16StringEncoding` or `NSUnicodeStringEncoding` can return incorrect results for Swift strings, including Swift values bridged to `NSString`. Avoid relying on these results until the defect is fixed (26.6-rc).
