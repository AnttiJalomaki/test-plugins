# Toolchain, Build, and Testing

## Choose a compatible Xcode host and device set

- Xcode 16.3 bundles the iOS and iPadOS 18.4 SDKs and requires macOS Sequoia 15.2 or later. Its device debugging floors are iOS and tvOS 15, watchOS 7, and visionOS (18.4).
- Xcode 16.4 bundles the iOS and iPadOS 18.5 SDKs and runs on macOS Sequoia 15.3 through macOS Tahoe 26.1 (18.5).
- Xcode 26 bundles Swift 6.2 and the version 26 SDKs for iOS, iPadOS, tvOS, watchOS, macOS Tahoe, and visionOS. It requires macOS Sequoia 15.6 or later; device debugging requires iOS or tvOS 15, watchOS 8, or visionOS (26.0).
- Xcode 26.6 bundles Swift 6.3 and the iOS, iPadOS, tvOS, watchOS, macOS, and visionOS 26.5 SDKs. It requires macOS Tahoe 26.2 or later (26.6-rc).
- Xcode 27 beta 2 bundles Swift 6.4 and the version 27 SDKs for iOS, iPadOS, tvOS, macOS, and visionOS. It requires macOS Tahoe 26.4 or later. Device debugging and Instruments require iOS or tvOS 17 and watchOS 10 or later (27.0-beta4).

## Meet upload requirements

Since April 28, 2026, App Store Connect uploads must be built with Xcode 26 or later and use a version 26 SDK for iOS, iPadOS, tvOS, visionOS, or watchOS (app-store-sdk-requirements).

## Build-setting migrations

### Swift explicit modules

Xcode 26 enables Swift explicit modules for Swift targets by default. Targets using a language mode earlier than Swift 5 or Swift/C++ interoperability are exceptions. When a serious compatibility issue blocks a build, temporarily set `SWIFT_ENABLE_EXPLICIT_MODULES=NO` while repairing the dependency or module graph (26.0).

### macOS and DriverKit architectures

The minimum supported macOS deployment target is 11.0. For macOS 27 and DriverKit 27 targets, `ARCHS_STANDARD` no longer includes `x86_64`; add it explicitly to `ARCHS` when a Universal binary remains necessary (27.0-beta4).

### Interface Builder

UIKit Interface Builder documents compile in `toolchain` mode by default, so CI build hosts no longer need a Simulator runtime solely for this compilation. To opt out during migration, set `IBC_COCOATOUCH_COMPILER_MODE=simulator` or pass `ibtool --cocoatouch-compiler-mode simulator` (27.0-beta4).

### hvf C API availability

Availability checks for hvf C APIs remain disabled unless this macro is defined before every hvf header (18.4):

```c
#define BUILD_FOR_APPLE_SDK 1
```

### Linker behavior

Using `LD_CLIENT_NAME` no longer needs the `ENABLE_DEBUG_DYLIB=NO` workaround for the missing debug-dylib runtime crash. The `-stack_size` linker flag can still fail for an app-bundle target under Xcode 16.3 (18.4).

## Swift package building

Xcode 26 previews a package builder shared with Swift Package Manager. It is planned to become the default. Enable it for compatibility testing with (26.0):

```sh
defaults write com.apple.dt.Xcode IDEEnableNewPackagePIFBuilder -bool YES
```

## Testing and diagnostics

### Swift Testing

Swift Testing can run exit tests around code that invokes `precondition()`, `fatalError()`, or otherwise terminates the test process. Save attachments to a selected directory with (26.0):

```sh
swift test --attachments-path <directory>
```

### Device diagnostics

`devicectl diagnose` gathers a sysdiagnose from the Mac and every available device by default; account for the expanded collection scope and output size (18.5):

```sh
devicectl diagnose
```

### Simulator and device-only checks

- Xcode 16.4 fixes an iOS 18.3 Simulator defect that made `NSURLSession` requests consistently time out (18.5).
- Safari extensions do not appear in iOS or visionOS Simulator, so test those extensions on a physical device (18.4).

### Xcode 27 limitations

- Parallel testing can significantly delay streamed `stdout` and `stderr` from multiple processes.
- Address Sanitizer builds produced with Xcode 26.4 or older might not launch on version 27 operating systems; use Xcode 26.5 or later to build those tests (27.0-beta4).

## Development integrations and previews

Xcode 26.6 supports the Agent Client protocol for coding-intelligence features. Its Preview Snapshot MCP tool can render light and dark appearances, portrait and landscape orientations, and type-size overrides (26.6-rc).
