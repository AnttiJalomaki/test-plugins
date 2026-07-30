# Graphics, Media, and Spatial APIs

## Nearby Interaction and broadcast capture

An app with an active Live Activity can perform Ultra Wideband ranging through
Nearby Interaction while it is in the background (18.4).

Broadcast Extensions have a higher per-process memory limit on iOS and iPadOS
18.5, permitting higher-quality capture and streaming when system resources
allow it (18.5). Treat the limit as resource-dependent rather than a fixed
budget guarantee.

## Metal and RealityKit

When encoding Metal 4 commands, put render and compute pipelines that support
indirect command buffers in the residency set. The driver does not currently
enforce the requirement, but code should not rely on that gap (26.0).

`Chart3D` uses RealityKit to render data and mathematical surfaces in three
dimensions on iOS 26, macOS 26, and visionOS 26 (26.6-rc).

The ShaderGraph `realitykit_hair_surfaceshader` node does not support
`DiffuseLightProbeGroupComponent`; a material using it might not respond to
diffuse light-probe-group lighting (27.0-beta4).

## SensorKit

The SensorKit PPG reader can return no samples when fetching data in the beta.
Handle empty results and revalidate the workaround on later seeds
(27.0-beta4).

## USDKit

USDKit cannot currently read or modify some USD attribute types. It also cannot
author array, vector, matrix, or quaternion values. Check attribute support
before selecting it for an authoring workflow (27.0-beta4).
