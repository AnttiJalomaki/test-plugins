---
name: android-sdk-knowledge-patch
description: Android SDK
version: null
license: MIT
metadata:
  author: Nevaberry
---


# Android SDK Knowledge Patch

Use this skill when changing an Android app's target or compile SDK, updating
Android Gradle Plugin, adopting recent platform APIs, or preparing a Google Play
submission. Start with the breaking-change checklist, then open the reference
that matches the code or policy being changed.

## Reference index

| Reference | Topics |
|---|---|
| [platform-ui-and-runtime.md](references/platform-ui-and-runtime.md) | Runtime behavior, windowing, predictive back, text, accessibility, input, broadcasts, system UI |
| [background-work-and-lifecycle.md](references/background-work-and-lifecycle.md) | Job quotas, abandoned jobs, background launches, audio, alarms, profiling |
| [networking-security-and-native-code.md](references/networking-security-and-native-code.md) | LAN permission, TLS, cleartext, intents, URI grants, page sizes, dynamic code, keys, signatures |
| [privacy-permissions-and-data.md](references/privacy-permissions-and-data.md) | Health sensors, selected media, SMS OTP, contacts, session location |
| [devices-bluetooth-camera-and-media.md](references/devices-bluetooth-camera-and-media.md) | Companion devices, Bluetooth, camera, codecs, Handoff, time-zone and NPU APIs |
| [agp-build-toolchain.md](references/agp-build-toolchain.md) | AGP and Gradle compatibility, public DSL, built-in Kotlin, R8, packaging, AGP 10 preparation |
| [play-publishing-policy.md](references/play-publishing-policy.md) | Submission target floors, existing-app discoverability, extensions and exemptions |

## Migration workflow

Before editing code or manifests, record these values from the project:

1. `compileSdk`, `targetSdk`, and `minSdk`.
2. AGP, Gradle, JDK, Build Tools, NDK, KGP, and KSP versions.
3. Native libraries and their ELF page alignment.
4. Background work, foreground-service, alarm, and background-audio paths.
5. Cross-app intents, URI sharing, LAN traffic, TLS, and cleartext exceptions.
6. Form factors distributed through Google Play.

Apply target-gated changes only when the app reaches the relevant target SDK.
Apply changes documented for all apps regardless of target as runtime changes.
Keep runtime OS checks separate from target-SDK checks.

## Highest-priority breaking changes

### When targeting API 36

- Remove the edge-to-edge opt-out and make every screen handle system insets.
- Migrate legacy Back handling; do not depend on `onBackPressed` or
  `KEYCODE_BACK` dispatch.
- Treat orientation and aspect restrictions as ineffective on large screens.
- Replace `BODY_SENSORS` permissions with granular health permissions and add
  the required privacy-rationale activity on mobile.
- Keep `JobParameters` alive through completion, call `jobFinished()`, and
  remove reliance on `setImportantWhileForeground()`.
- Audit fixed-rate tasks that expect every missed invocation to replay.
- Treat ordered-broadcast priority as process-local.
- Rebuild native libraries for 16 KB page alignment even when compatibility
  mode is available.
- Treat app-owned picker media as revocable selected-media access.

Open the platform, background-work, privacy, and networking references for the
full conditions and migration details.

### When targeting API 37

- Declare and request `ACCESS_LOCAL_NETWORK` for direct LAN discovery or
  connections, or use a system-mediated picker.
- Make native files read-only before passing them to `System.load()`.
- Remove reflection or JNI code that mutates `static final` fields.
- Remove reflection against private `MessageQueue` internals.
- Require RFCOMM read loops to stop when `read()` returns `-1`.
- Move `ContactsContract.Data` account-field access to a `RawContacts` join.
- Provide a qualifying foreground service before background audio playback or
  focus requests.
- Replace broad `IntentSender` background-launch modes with a granular mode.
- Expect certificate transparency and ECH-capable TLS behavior.
- Account for the Android Keystore key cap and inspect numeric error codes.
- Declare the NPU feature before direct NPU access.

### Runtime changes independent of target SDK

- Diagnose memory-limiter exits through `ApplicationExitInfo`; do not mistake
  them for ordinary low-memory process death.
- Replace cross-profile loopback communication because Android 17 blocks it.
- Move OTP flows to SMS Retriever or SMS User Consent when immediate SMS
  visibility is required.
- Do not assume activity recreation or IME visibility restoration for the
  affected configuration changes.
- Test custom notification views against the enforced size limits.

## AGP 9 migration quick reference

### Toolchain matrix

| AGP | Supported SDK ceiling | Required Gradle | Required JDK |
|---|---:|---:|---:|
| 9.0 | 36.1 | 9.1.0 | 17 |
| 9.2 | 37.0 | 9.4.1 | 17 |
| 9.3 | 37 | 9.5.0 | 17 |

These releases use Build Tools 36.0.0 and default to NDK
28.2.13676358 (`r28c`). Do not select a compile SDK above the AGP release's
supported ceiling.

### Required build-logic migrations

Replace legacy variant and extension access with the public APIs:

```kotlin
androidComponents {
    beforeVariants(selector().withBuildType("debug")) { it.enable = false }
    onVariants { variant ->
        // Configure lazy variant artifacts and sources here.
    }
}
```

- Use `androidComponents.onVariants` instead of `applicationVariants` and its
  siblings.
- Use `androidComponents.beforeVariants` instead of `variantFilter`.
- Use `androidComponents.sdkComponents` for SDK paths.
- Use Gradle-managed devices instead of custom test providers.
- Use the Sources API for generated-source providers.
- Move bytecode transforms to `Instrumentation`.

`android.newDsl=false` is only a temporary AGP 9 compatibility switch. It is
not an AGP 10 migration strategy.

### Built-in Kotlin

Remove `org.jetbrains.kotlin.android` and `kotlin-android` from Android modules
when using AGP 9 built-in Kotlin. Check KGP and KSP constraints before the
upgrade. For multiplatform code, use the Android Gradle Library Plugin for KMP
and place the Android application in a separate subproject.

### Default changes to audit

- unique library package names;
- AndroidX dependency defaults;
- non-final application `R` fields;
- target SDK defaulting to compile SDK;
- `resValues` disabled unless enabled per module;
- only the tested build type receiving a unit-test component;
- `AndroidJUnitRunner` as the on-device test default;
- dependency constraints disabled by default outside application device tests.

### R8 and resource shrinking

Missing keep files fail the build. Integrated optimized resource shrinking is
mandatory. Explicitly retain constructors under strict full-mode semantics:

```proguard
-keep class com.example.EntryPoint { <init>(); }
```

Do not publish global optimization or obfuscation options in library consumer
rules. Name runtime-invisible annotation attributes explicitly on AGP 9.2.
Use the R8 Configuration Analyzer task on AGP 9.3 before a full release build:

```shell
./gradlew :app:analyzeReleaseR8Config
```

See the build-toolchain reference before changing desugaring, mapping IDs,
Kotlin null-check processing, optimization blocks, or plugin APIs.

## Security and permission quick reference

### Local network

Use the Android 16 compatibility change to test denial and revocation before a
target-SDK migration:

```shell
adb shell am compat enable RESTRICT_LOCAL_NETWORK com.example.app
```

For API 37 targets, add the runtime permission:

```xml
<uses-permission android:name="android.permission.ACCESS_LOCAL_NETWORK" />
```

Test in-process TCP, UDP, multicast, broadcast, and native sockets separately
from out-of-process discovery APIs.

### Intent launch hardening

Keep nested-intent launch protection enabled. Use
`removeLaunchSecurityProtection()` only around a verified flow that cannot be
migrated immediately. Consider strict incoming-intent matching:

```xml
<application android:intentMatchingFlags="enforceIntentFilter" />
```

Use StrictMode to find implicit URI grants and add explicit read or write flags
according to the receiving operation.

### Network Security Configuration

Move cleartext domain exceptions into Network Security Configuration. Keep
`usesCleartextTraffic` as a temporary companion only when `minSdk` is below 24.
Use `<domainEncryption>` when ECH must be controlled globally or per domain.

## Platform API quick reference

- `JobScheduler.getPendingJobReasonsHistory()` explains why pending jobs have
  not run.
- `PhotoPickerUiCustomizationParams` supports a portrait 9:16 picker grid.
- `CameraCaptureSession.updateOutputConfigurations()` changes use cases without
  closing the session.
- `AlarmManager.setExactAndAllowWhileIdle()` has an `OnAlarmListener` overload
  for in-process callbacks.
- `ProfilingManager` supports cold-start, OOM, and excessive-CPU-kill triggers.
- `RemoteViews.setViewPadding()` accepts complex DP/SP units, and
  `OPTION_APPWIDGET_DISPLAY_ID` identifies the host display.
- `TextAttribute` and `AccessibilityEvent` expose composition and candidate
  selection metadata for complex IME workflows.
- Companion Device Manager supports Handoff, Medical Device and Fitness Tracker
  profiles, and association-time nearby permissions.

## Publishing check

Before release, open the Play policy reference and evaluate each distributed
form factor independently. Submission eligibility and discoverability of an
existing app use different target floors. If an extension is needed, request it
through the app's Play Console warning before the stated deadline.

## Verification checklist

- Run affected compatibility flags on physical or representative virtual
  devices.
- Exercise background work after the app loses visibility.
- Test Back, edge-to-edge, rotation, resize, desktop, and IME transitions.
- Test LAN permission denial, revocation, and an existing nearby-device grant.
- Verify URI access in every share and image-capture path.
- Load every native library from its final on-device location.
- Test Bluetooth reads, bond loss, and companion timeout outcomes.
- Build minified release variants and inspect R8 configuration and mapping IDs.
- Validate Play target requirements per form factor, not just per package.
