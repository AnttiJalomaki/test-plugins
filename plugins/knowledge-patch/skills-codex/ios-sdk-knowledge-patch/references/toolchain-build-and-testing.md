# Toolchain, Build, and Testing

## Select a compatible Xcode, host, and SDK

| Xcode | Bundled iOS/iPadOS SDK | Build-host requirement | Device-debugging floor |
| --- | --- | --- | --- |
| 16.3 | 18.4 | macOS Sequoia 15.2 or later | iOS/tvOS 15, watchOS 7, and visionOS |
| 16.4 | 18.5 | macOS Sequoia 15.3 through macOS Tahoe 26.1 | Use the platform support supplied by Xcode |
| 26 | 26, with Swift 6.2 and the corresponding tvOS, watchOS, macOS Tahoe, and visionOS SDKs | macOS Sequoia 15.6 or later | iOS/tvOS 15, watchOS 8, and visionOS |

These requirements are tied to their respective SDK releases (18.4, 18.5,
and 26.0). A deployment target does not relax the Xcode host requirement.

## Handle Simulator and device-only behavior

Safari extensions do not appear in the iOS or visionOS Simulator. Test those
extensions on a physical device. (18.4)

Xcode 16.4 fixes the iOS 18.3 Simulator-runtime defect that made every
`NSURLSession` request time out. When reproducing that symptom, distinguish the
selected Simulator runtime from the SDK used to build the app. (18.5)

## Collect diagnostics from all devices

`devicectl diagnose` now collects a sysdiagnose from the Mac and all available
devices by default; no additional device-selection argument is required for
that scope. Expect a larger collection when several devices are connected.
(18.5)

```sh
devicectl diagnose
```

## Revisit linker workarounds

Using `LD_CLIENT_NAME` no longer requires `ENABLE_DEBUG_DYLIB=NO` to avoid the
missing debug-dylib runtime crash in Xcode 16.3. Remove that workaround if it
was added only for this issue. The `-stack_size` linker flag can still fail for
an app-bundle target, so do not treat the debug-dylib fix as resolving that
separate failure. (18.4)

## Account for Swift explicit modules

Xcode 26 enables Swift explicit modules for Swift targets by default. The
default does not apply to targets using a language version older than Swift 5
or to targets using Swift/C++ interoperability. If a severe compatibility
failure cannot be addressed immediately, opt out temporarily: (26.0)

```text
SWIFT_ENABLE_EXPLICIT_MODULES=NO
```

## Try the preview package builder deliberately

Xcode 26 includes a preview of a package-build implementation shared with
Swift Package Manager. It is planned to become the default later, so enable it
in a controlled build or CI lane before relying on it: (26.0)

```sh
defaults write com.apple.dt.Xcode IDEEnableNewPackagePIFBuilder -bool YES
```

Because this is a preview switch, keep a lane using the established builder
until compatibility is established.

## Use Swift Testing exit tests and attachment paths

Swift Testing can exercise code paths that terminate the test process through
`precondition()`, `fatalError()`, or another exit. It also supports directing
attachments to an explicit directory: (26.0)

```sh
swift test --attachments-path ./test-artifacts
```

Choose a path retained by CI if attachments are needed for failure diagnosis.
