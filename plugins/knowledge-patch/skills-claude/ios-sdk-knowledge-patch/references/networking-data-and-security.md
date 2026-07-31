# Networking, Data, and Security

Use this reference for HTTP loading, Core Data migration and concurrency,
date parsing, cross-process primitives, VPN profiles, and transport security.

## URLSession HTTP Loading Mode

iOS 18.4 (`18.4`) adds a newer HTTP loading mode. Opt a configuration into it
by setting `usesClassicLoadingMode` to `false`:

```swift
let configuration = URLSessionConfiguration.default
configuration.usesClassicLoadingMode = false
let session = URLSession(configuration: configuration)
```

Classic loading remains the default for now, but the newer mode is planned to
become the default in a future release. Test protocol, proxy, authentication,
cache, and timing-sensitive behavior before adopting it broadly.

## iOS 18.3 Simulator URLSession Timeouts

Xcode 16.4 (`18.5`) fixes an iOS 18.3 Simulator runtime defect that caused
`NSURLSession` requests to time out and fail consistently. If this exact symptom
appears with the older toolchain/runtime combination, update Xcode before
rewriting networking code or weakening timeout policy.

## Core Data Concurrency Imports

The iOS 26 SDK (`26.0`) imports the principal Core Data concurrency types as
follows:

- `NSManagedObject` is nonisolated and non-`Sendable`.
- `NSManagedObjectContext` is nonisolated and `Sendable`.
- The `perform` and `performBlock` families accept `Sendable` closures.

Rebuilding can expose new concurrency warnings. Keep each managed object inside
its context's scope instead of sending it across isolation boundaries. During
testing, launch with this argument to catch confinement violations:

```text
-com.apple.CoreData.ConcurrencyDebug 1
```

## Removed Core Data Ubiquity Options

The iOS and macOS 26 SDKs (`26.0`) reject these legacy store options at build
time; binaries built with older SDKs log warnings:

- `NSPersistentStoreUbiquitousContentNameKey`
- `NSPersistentStoreUbiquitousContentURLKey`
- `NSPersistentStoreUbiquitousPeerTokenOption`
- `NSPersistentStoreRemoveUbiquitousMetadataOption`
- `NSPersistentStoreUbiquitousContainerIdentifierKey`
- `NSPersistentStoreRebuildFromUbiquitousContentOption`

Removing the options preserves a local store but stops its synchronization.
Move synchronization to `NSPersistentCloudKitContainer` or SwiftData rather
than silently accepting a local-only store.

## ISO-8601 Fractional Seconds

In the iOS 26 SDK (`26.0`), `ISO8601FormatStyle` accepts fractional seconds
regardless of the `includingFractionalSeconds` setting. Do not use that setting
as strict input rejection or validation policy; validate separately if a wire
format must prohibit fractions.

## Team-Scoped POSIX Named Semaphores

On iOS 26 (`26.0`), processes signed with a Team ID entitlement cannot use
`sem_open` or `sem_unlink` to observe a named semaphore created by another
development team. Design named-semaphore coordination within one signing team,
and use another supported IPC mechanism for cross-team communication.

## IKEv2 Cryptography Removal

iOS 26 (`26.0`) removes IKEv2 support for:

- DES and 3DES
- SHA1-96 and SHA1-160
- Diffie-Hellman groups below 14

Update both the client VPN profile and the VPN server.

## TLS 1.2 Minimum

For apps linked on or after iOS 26 or macOS 26 (`26.0`), `URLSession` and the
Network framework default to TLS 1.2 rather than TLS 1.0 as their minimum.

When a legacy endpoint truly requires an older protocol, make the exception
explicit with one of these APIs:

- `URLSessionConfiguration.tlsMinimumSupportedProtocolVersion`
- `sec_protocol_options_set_min_tls_protocol_version`

Prefer upgrading the endpoint. Keep any lower minimum narrowly scoped and test
the exact linked binary because this is linked-on behavior.
