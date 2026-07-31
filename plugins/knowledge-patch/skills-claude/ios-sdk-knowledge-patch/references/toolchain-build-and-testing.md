# Toolchain, Build, and Testing

Use this reference to select an Xcode/host combination, diagnose device and
Simulator behavior, manage linker and module changes, and adopt build and test
features.

## Selecting Xcode and the Host

Xcode 16.3, associated with the 18.4 SDK batch (`18.4`), bundles the iOS and
iPadOS 18.4 SDKs and requires macOS Sequoia 15.2 or later. Its on-device
debugging floors are iOS and tvOS 15, watchOS 7, and visionOS.

Xcode 16.4, associated with the 18.5 SDK batch (`18.5`), bundles the iOS and
iPadOS 18.5 SDKs and supports hosts from macOS Sequoia 15.3 through macOS Tahoe
26.1.

Xcode 26, associated with the stable 26.0 SDK batch (`26.0`), bundles Swift 6.2
and the version 26 SDKs for iOS, iPadOS, tvOS, watchOS, macOS Tahoe, and
visionOS. It requires macOS Sequoia 15.6 or later. Its on-device debugging
floors are iOS and tvOS 15, watchOS 8, and visionOS.

Distinguish these three constraints when troubleshooting: the host floor, the
SDK bundled for compilation, and the device OS floor supported for debugging.

## Simulator Safari Extension Limitation

Safari extensions do not appear in iOS or visionOS Simulator with Xcode 16.3
(`18.4`). Test extension discovery and behavior on a physical device for those
platforms.

## Linker and Debug Dylib Behavior

For Xcode 16.3 (`18.4`), using `LD_CLIENT_NAME` no longer needs the
`ENABLE_DEBUG_DYLIB=NO` workaround for a missing debug-dylib runtime crash.
Remove the workaround unless another independently verified issue still needs
it.

The `-stack_size` linker flag can still fail for an app-bundle target in Xcode
16.3. If the flag is essential, reproduce the failure in a minimal app-bundle
target and avoid assuming the failure comes from application startup code.

## Device Diagnostics Collection

In Xcode 16.4 (`18.5`), this command collects a sysdiagnose from the Mac and
from every available device by default:

```sh
devicectl diagnose
```

No additional per-device argument is required. Expect a broader collection,
more output, and potentially longer collection time than a Mac-only diagnostic.

## Metal 4 Indirect Command Buffer Residency

When using Metal 4 command encoders from the version 26 SDK (`26.0`), include
render and compute pipelines that support indirect command buffers in the
residency set. The driver may not currently enforce this requirement, but code
must not depend on that enforcement gap.

## Preview Swift Package Builder

Xcode 26 (`26.0`) previews a package-build implementation shared with Swift
Package Manager. It is planned to become the default later. Enable it for
targeted compatibility testing with:

```sh
defaults write com.apple.dt.Xcode IDEEnableNewPackagePIFBuilder -bool YES
```

Because this is a preview opt-in, compare build graphs and results with the
current default before enabling it for a team-wide workflow.

## Swift Testing Exit Tests and Attachments

Swift Testing in Xcode 26 (`26.0`) supports exit tests for code expected to call
`precondition()`, `fatalError()`, or otherwise terminate the test process.
Prefer an exit test over allowing intentional termination to abort the whole
suite.

Choose the output directory for Swift Testing attachments with:

```sh
swift test --attachments-path <directory>
```

Create or clean the directory according to the surrounding CI job's artifact
policy before invoking the command.
