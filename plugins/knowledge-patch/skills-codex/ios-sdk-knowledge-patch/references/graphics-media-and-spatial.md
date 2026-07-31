# Graphics, Media, and Spatial APIs

## Range in the background with Nearby Interaction

An app with an active Live Activity can perform Ultra Wideband ranging through
Nearby Interaction while it is in the background. (18.4)

## Use the higher Broadcast Extension memory limit carefully

Broadcast Extensions receive a higher per-process memory limit on iOS and
iPadOS 18.5. This can support higher-quality capture and streaming, but the
actual headroom still depends on system resources. Continue responding to
memory pressure and test on representative devices. (18.5)

## Migrate legacy Push to Talk entitlement use

Apps built with the iOS 26 SDK or later cannot use
`com.apple.developer.pushkit.unrestricted-voip.ptt`. Migrate Push to Talk
features to the Push to Talk framework introduced in iOS 16, and remove the
legacy entitlement from new-SDK builds. (26.0)

## Declare Metal 4 indirect-command-buffer residency

When encoding with Metal 4, add render and compute pipelines that support
indirect command buffers to the residency set even though the Metal driver does
not currently enforce the requirement. (26.0)
