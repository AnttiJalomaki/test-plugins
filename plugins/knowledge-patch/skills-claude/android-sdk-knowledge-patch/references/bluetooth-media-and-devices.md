# Bluetooth, Media, and Device Integration

Use this reference for companion association, wireless links, camera and media
features, background audio, location affordances, and cross-device APIs. The
API-specific source batches are identified as `api-36` and `api-37`.

## Companion association and Bluetooth

### Discovery timeout result (`api-36`)

On Android 16, companion-device discovery timeouts no longer return
`RESULT_DISCOVERY_TIMEOUT` directly. The system presents a timeout dialog and,
after dismissal, reports `RESULT_USER_REJECTED`. Association code must not
interpret that result only as an explicit rejection.

### Authentication failure and retained bonds (`api-36`)

Android 16 disconnects a device after failed authentication, retains the local
bond, and asks the user to pair again. It no longer silently removes the bond
and immediately starts pairing.

API 36-targeted apps can observe `ACTION_KEY_MISSING` and
`ACTION_ENCRYPTION_CHANGE`, but must tolerate devices whose OEM software omits
these broadcasts. For a managed companion association, remove the bond with
`CompanionDeviceManager.removeBond(int)` when policy requires it.

### RFCOMM EOF (`api-37`)

For API 37-targeted apps, an RFCOMM `BluetoothSocket` input stream returns
`-1` when the socket closes or the connection drops. Read loops must stop on
`-1`, not only on `IOException`.

### Autonomous bond repair (`api-37`)

Android 17 can re-pair in the background after bond loss. Keys are replaced
only after a successful connection with equal or stronger security.
`ACTION_PAIRING_REQUEST` includes `EXTRA_PAIRING_CONTEXT`, and
`ACTION_KEY_MISSING` is delayed until autonomous repair fails.

### New companion profiles (`api-37`)

Companion Device Manager adds Medical Device and Fitness Tracker profiles.
`setExtraPermissions()` can fold nearby-device grants into the association
dialog.

## Audio and media

### Lifecycle-gated background audio (`api-37`)

On Android 17, invalid background playback and volume operations fail
silently, and audio-focus requests return `AUDIOFOCUS_REQUEST_FAILED`.
API 37-targeted apps also need a foreground service with while-in-use
capability for background audio.

The exception is an app that holds exact-alarm permission and is using a
`USAGE_ALARM` stream.

### Codecs and output categories (`api-37`)

Android 17 adds:

- VVC/H.266 platform support.
- Constant-quality video through `MediaRecorder.setVideoEncodingQuality()`.
- The `c2.android.xheaac.encoder` software encoder with loudness metadata.
- `AudioDeviceInfo.TYPE_BLE_HEARING_AID`.

## Photo Picker and camera

### Custom Photo Picker layout (`api-37`)

`PhotoPickerUiCustomizationParams` changes the picker grid from its default
square cells to a 9:16 portrait aspect ratio.

### Camera formats and session updates (`api-37`)

Android 17 adds `ImageFormat.RAW14`, vendor-defined extension modes
discoverable with `isExtensionSupported(int)`, camera device-type APIs, and
`CameraCaptureSession.updateOutputConfigurations()`. The session method
changes use cases without closing the capture session.

## User-mediated access and device features

### Session location button (`api-37`)

Jetpack can embed a system-rendered location button that grants precise
location for the current session without a permission dialog. Declare
`USE_LOCATION_BUTTON`.

### Handoff, time-zone offsets, and NPU declaration (`api-37`)

Android 17 adds cross-device Handoff through `CompanionDeviceManager` and
`ACTION_TIMEZONE_OFFSET_CHANGED` for DST or other offset-only changes.
API 37-targeted apps must declare `FEATURE_NEURAL_PROCESSING_UNIT` before
accessing an NPU directly.
