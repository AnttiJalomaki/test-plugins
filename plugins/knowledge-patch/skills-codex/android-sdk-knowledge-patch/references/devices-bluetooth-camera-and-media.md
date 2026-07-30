# Devices, Bluetooth, Camera, and Media

## Companion-device association

### Discovery timeouts

On Android 16, a companion-device discovery timeout no longer directly returns `RESULT_DISCOVERY_TIMEOUT`. The system displays a timeout dialog and, after the user dismisses it, reports `RESULT_USER_REJECTED`. Do not interpret that result exclusively as an explicit rejection. (`api-36`)

### Profiles and association permissions

Android 17 adds Medical Device and Fitness Tracker profiles to Companion Device Manager. `setExtraPermissions()` can fold nearby-device grants into the association dialog.

### Cross-device handoff

Android 17 adds cross-device Handoff through `CompanionDeviceManager`.

## Bluetooth behavior

### Authentication and bond loss on Android 16

When authentication fails, Android 16 disconnects the device, retains the local bond, and asks the user to pair again instead of silently deleting the bond and starting pairing. API 36 targets can observe `ACTION_KEY_MISSING` and `ACTION_ENCRYPTION_CHANGE`, but must tolerate OEMs that omit those broadcasts. For a managed companion association, `CompanionDeviceManager.removeBond(int)` can remove the bond.

### Autonomous repair on Android 17

Android 17 can re-pair in the background after bond loss. It replaces keys only after a successful connection with equal or stronger security. `ACTION_PAIRING_REQUEST` adds `EXTRA_PAIRING_CONTEXT`, and `ACTION_KEY_MISSING` is delayed until autonomous re-pairing fails.

### RFCOMM EOF

For API 37 targets, an RFCOMM `BluetoothSocket` input stream returns `-1` when the socket closes or the connection drops. Every read loop must test for `-1` rather than relying only on `IOException`. (`api-37`)

## Camera and media

### Camera formats and session updates

Android 17 adds:

- `ImageFormat.RAW14`;
- implementation-defined extension modes discoverable with `isExtensionSupported(int)`;
- camera device-type APIs;
- `CameraCaptureSession.updateOutputConfigurations()` for changing use cases without closing the session.

### Codecs, quality, and output devices

Android 17 adds VVC/H.266 platform support, constant-quality video through `MediaRecorder.setVideoEncodingQuality()`, the `c2.android.xheaac.encoder` software encoder with loudness metadata, and `AudioDeviceInfo.TYPE_BLE_HEARING_AID`.

## Other device integration

Android 17 adds `ACTION_TIMEZONE_OFFSET_CHANGED` for daylight-saving and other offset-only time-zone changes.

An API 37-targeted app must declare `FEATURE_NEURAL_PROCESSING_UNIT` before directly accessing an NPU.
