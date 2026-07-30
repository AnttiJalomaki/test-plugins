# Toolchain, Build, and Testing

## Xcode hosts, SDKs, and device support

- Xcode 16.3 bundles the iOS and iPadOS 18.4 SDKs and requires macOS Sequoia
  15.2 or later. Device debugging supports iOS and tvOS 15 or later, watchOS 7
  or later, and visionOS (18.4).
- Xcode 16.4 bundles the iOS and iPadOS 18.5 SDKs and runs on macOS Sequoia
  15.3 through macOS Tahoe 26.1 (18.5).
- Xcode 26 bundles Swift 6.2 and the iOS, iPadOS, tvOS, watchOS, macOS Tahoe,
  and visionOS 26 SDKs. It requires macOS Sequoia 15.6 or later; device
  debugging supports iOS and tvOS 15 or later, watchOS 8 or later, and
  visionOS (26.0).
- Xcode 26.6 bundles Swift 6.3 and the iOS, iPadOS, tvOS, watchOS, macOS, and
  visionOS 26.5 SDKs. It requires macOS Tahoe 26.2 or later (26.6-rc).
- Xcode 27 beta 2 includes Swift 6.4 and the iOS, iPadOS, tvOS, macOS, and
  visionOS 27 SDKs. It requires macOS Tahoe 26.4 or later. Device debugging
  and Instruments require iOS or tvOS 17, watchOS 10, or later (27.0-beta4).

## Build settings and compiler behavior

### hvf availability checks

Availability checking for hvf C APIs is off unless `BUILD_FOR_APPLE_SDK` is
defined before any hvf header (18.4):

```c
#define BUILD_FOR_APPLE_SDK 1
```

### Linker and debug dylibs

Using `LD_CLIENT_NAME` no longer needs the `ENABLE_DEBUG_DYLIB=NO` workaround
for the missing debug-dylib runtime crash. The `-stack_size` linker flag can
still fail for an app-bundle target in Xcode 16.3 (18.4).

### Swift explicit modules

Xcode 26 enables Swift explicit modules for Swift targets by default. Targets
using a pre-Swift-5 language mode or Swift/C++ interop are excluded. For a
severe compatibility failure, temporarily set (26.0):

```text
SWIFT_ENABLE_EXPLICIT_MODULES=NO
```

### Swift package build implementation

Xcode 26 previews a package builder shared with Swift Package Manager. Enable
it for compatibility testing with the following command; it is planned to
become the default later (26.0):

```sh
defaults write com.apple.dt.Xcode IDEEnableNewPackagePIFBuilder -bool YES
```

### Interface Builder toolchain mode

UIKit Interface Builder documents compile in `toolchain` mode by default,
removing the simulator-runtime requirement from build servers. During the
transition, opt out with `IBC_COCOATOUCH_COMPILER_MODE=simulator` or (27.0-beta4):

```sh
ibtool --cocoatouch-compiler-mode simulator
```

### macOS deployment and architecture

The minimum supported macOS deployment target is 11.0. For macOS 27 and
DriverKit 27 targets, `ARCHS_STANDARD` omits `x86_64`; explicitly add it to
`ARCHS` when producing a Universal binary (27.0-beta4).

## Testing and diagnostics

### Swift Testing

Swift Testing supports exit tests for code that invokes `precondition()`,
`fatalError()`, or otherwise terminates the test process. Select the attachment
output directory with (26.0):

```sh
swift test --attachments-path <directory>
```

### Device diagnostics

`devicectl diagnose` collects a sysdiagnose from the Mac and every available
device by default, so review the enlarged data and time scope before running it
in automation (18.5):

```sh
devicectl diagnose
```

### Simulator and device-only testing

- Safari extensions do not appear in the iOS or visionOS Simulator. Test them
  on a device (18.4).
- Xcode 16.4 fixes the iOS 18.3 Simulator runtime defect that made every
  `NSURLSession` request time out (18.5).
- Streaming `stdout` and `stderr` from multiple processes can be substantially
  delayed during parallel testing on the Xcode 27 toolchain (27.0-beta4).
- Address Sanitizer binaries built with Xcode 26.4 or older might not launch on
  27.0 operating systems. Build those tests with Xcode 26.5 or later
  (27.0-beta4).

## Coding intelligence and previews

Xcode supports the Agent Client protocol for coding-intelligence features
(26.6-rc). The Preview Snapshot MCP tool can render light and dark appearances,
portrait and landscape orientations, and type-size overrides (26.6-rc).
