---
name: android-sdk-knowledge-patch
description: Android SDK
version: null
license: MIT
metadata:
  author: Nevaberry
---


# Android SDK Compatibility Guide

Use this skill when changing an Android application's target SDK, adopting
Android Gradle Plugin 9.x, preparing for the AGP 10 build model, or checking
Google Play target-API eligibility. Start from the project's manifest and
Gradle files, then load the reference that matches the code being changed.

## Reference index

| Reference | Topics |
| --- | --- |
| [references/runtime-ui-and-scheduling.md](references/runtime-ui-and-scheduling.md) | Jobs, executors, broadcasts, activities, back, layouts, text, IME, memory, alarms, notifications, widgets, desktop windows |
| [references/security-networking-and-data.md](references/security-networking-and-data.md) | Page size, permissions, intents, LAN access, TLS, contacts, URI grants, Keystore, signing, HPKE |
| [references/bluetooth-media-and-devices.md](references/bluetooth-media-and-devices.md) | Companion devices, Bluetooth, media, audio, cameras, Photo Picker, location, cross-device APIs |
| [references/agp-toolchain-and-apis.md](references/agp-toolchain-and-apis.md) | AGP/Gradle/JDK floors, built-in Kotlin, public DSL, Variant API, KMP, defaults, removed APIs, AGP 10 staging |
| [references/r8-packaging-and-testing.md](references/r8-packaging-and-testing.md) | Shrinker rules, R8 options, resource shrinking, packaging removals, shaders, fused AARs, test aggregation |
| [references/play-target-api-policy.md](references/play-target-api-policy.md) | Submission floors, existing-app availability, extensions, exemptions |

## Triage workflow

1. Read `compileSdk`, `targetSdk`, `minSdk`, the AGP version, the Gradle
   wrapper, the Java toolchain, Kotlin plugins, and KSP configuration.
2. Separate target-gated behavior from behavior that affects every app on the
   new OS. Do not assume an older `targetSdk` avoids an all-app change.
3. Search manifests and code for permissions, intent flags, configuration
   changes, background work, foreground services, native libraries, and
   private-API reflection relevant to the upgrade.
4. Search build logic for legacy DSL objects, eager variant APIs, removed
   Gradle properties, old Transform hooks, and consumer shrinker rules.
5. Apply the smallest compatible migration, then test on the relevant OS,
   screen class, input device, profile, and background state.
6. Check distribution eligibility separately from runtime compatibility.
   Google Play policy has different floors for updates and discoverability.

## Highest-risk target SDK changes

### Targeting API 36

- Edge-to-edge enforcement cannot be disabled on Android 16. Remove reliance
  on `windowOptOutEdgeToEdgeEnforcement` and make every screen consume insets.
- Predictive back becomes the normal path. Move away from legacy
  `onBackPressed` and `KEYCODE_BACK`; use supported back callbacks.
- Large-screen displays at `sw600dp` or above ignore most orientation,
  resizability, and aspect-ratio restrictions. Treat the temporary manifest
  opt-out only as migration time.
- Job runtime quotas also cover work started while visible and continued after
  visibility loss, active-bucket work, and jobs beside foreground services.
- Health sensor access moves from broad body-sensor permissions to granular
  health permissions, including a background health-data permission.
- Cross-app intent hardening changes nested-intent launches. Never remove the
  launch protection unless a reviewed, legitimate flow requires it.
- App-owned media is no longer guaranteed under selected-media access: the
  user can deselect it and revoke access immediately.

### Targeting API 37

- Declare and request `ACCESS_LOCAL_NETWORK` for direct LAN discovery and
  connections. A system device picker is the permissionless alternative.
- Certificate transparency becomes automatic, and supported TLS stacks use
  Encrypted Client Hello unless Network Security Configuration disables it.
- Native code loaded dynamically must be read-only before `System.load()`.
- Background playback and volume operations require a valid lifecycle;
  target-gated playback also needs the appropriate foreground service.
- Immediate access to ordinary OTP-bearing SMS is removed, and WebOTP is
  withheld from non-recipients. Prefer SMS Retriever or SMS User Consent.
- `BluetoothSocket` RFCOMM reads report closure as `-1`; terminate read loops
  on EOF instead of waiting only for `IOException`.
- Reflection cannot mutate `static final` fields, and JNI setters attempting
  it crash the app. Remove this mechanism rather than catching around it.
- Activity recreation defaults change for several configurations. Explicitly
  request recreation or handle updates in place.

## AGP 9 migration essentials

### Align the toolchain first

Choose a mutually supported AGP, Gradle, SDK, and JDK set before editing DSL
code. AGP 9.x requires JDK 17; individual minor releases require specific
Gradle versions and differ in API-level support. See the toolchain matrix in
[references/agp-toolchain-and-apis.md](references/agp-toolchain-and-apis.md).

### Replace legacy variant access

Use `androidComponents` and lazy providers. A common variant-filter migration
is:

```kotlin
androidComponents {
    beforeVariants(selector().withBuildType("debug")) {
        it.enable = false
    }
}
```

Replace these families together:

| Legacy mechanism | Supported direction |
| --- | --- |
| `applicationVariants` and siblings | `androidComponents.onVariants` |
| `variantFilter` | `androidComponents.beforeVariants` |
| SDK path getters | `androidComponents.sdkComponents` |
| Transform API | `variant.instrumentation.transformClassesWith` |
| Eager generated-source wiring | `variant.sources.*.addGeneratedSourceDirectory` |
| Direct task access | Lazy providers and `Variant.artifacts` |

`android.newDsl=false` is only a temporary AGP 9 compatibility switch. Code
that still needs it is not ready for AGP 10.

### Remove redundant Kotlin plugin application

Built-in Kotlin is enabled by default. Android modules should not also apply
`org.jetbrains.kotlin.android` or `kotlin-android`. Check the carried Kotlin
and KSP versions before forcing another version, and use the documented
top-level classpath or opt-out path only when the project truly requires it.

For Kotlin Multiplatform, do not combine the standard Android application or
library plugin with the new DSL in the same KMP subproject. Use the Android
Gradle Library Plugin for the KMP library and place an Android application in
a separate subproject.

### Audit changed defaults

- Give every Android library a unique namespace.
- Set `targetSdk` explicitly if inheriting `compileSdk` is unintended.
- Expect non-final application `R` fields and AndroidX dependencies.
- Enable `resValues`, AIDL, or RenderScript only in modules that need them.
- Configure unit-test components intentionally; only the tested build type is
  created by default.
- Do not assume dependency constraints propagate with the old default.

## Shrinker and packaging checks

Missing keep files are build errors, optimized resource shrinking is active,
and full-mode rules do not infer constructor retention:

```proguard
-keep class A { <init>(); }
```

Consumer rules must not publish global options such as `-dontoptimize` or
`-dontobfuscate`. Keep runtime-invisible annotation attributes by naming all
three attributes explicitly, and inspect Kotlin null-check rewriting before
changing its default message-removal behavior.

If an obfuscated stack trace carries `r8-map-id-<MAP_ID>` in `SourceFile`, use
that full mapping hash to select the mapping file. A custom source-file rename
overrides this marker.

AGP 9 removes density-split APKs, embedded Wear app packaging, and several
report tasks. Prefer app bundles for density delivery, publish Wear artifacts
separately, and use the available aggregation dashboards or R8 analyzer for
the reports they actually provide.

## Manifest and network checklist

For an API-level migration, review at least:

```xml
<uses-permission android:name="android.permission.ACCESS_LOCAL_NETWORK" />
```

- Granular health permissions and the required privacy-rationale activity.
- `USE_LOCATION_BUTTON` when using the system session-location affordance.
- `FEATURE_NEURAL_PROCESSING_UNIT` before direct NPU access where required.
- Explicit URI read/write grants on send and image-capture intents.
- Network Security Configuration for cleartext exceptions and ECH policy.
- `android:recreateOnConfigChanges` and IME restoration behavior.
- Large-screen orientation and resizability assumptions.
- The monochrome adaptive-icon layer used for launcher theming.

## Validation commands and probes

Use these only against a disposable test device or emulator and substitute the
real package, process, UID, and limits:

```shell
adb shell am compat enable RESTRICT_LOCAL_NETWORK com.example.app
adb reboot
adb shell am memory-limiter status
adb shell am memory-limiter manual PID LIMIT
adb shell am memory-limiter ignore UID
```

Also exercise:

- Job stop reasons and `getPendingJobReasonsHistory()` under background and
  foreground-service transitions.
- Back gestures in gesture navigation and long-press back in 3-button mode.
- `sw600dp` layouts in full-screen and multi-window modes.
- LAN permission denial and revocation, including native sockets.
- Closed RFCOMM streams, autonomous Bluetooth re-pairing, and missing OEM
  broadcasts.
- Custom notification layouts at platform-enforced size limits.
- Native libraries on 16 KB-page devices without depending on compatibility
  mode.

## Distribution decision

Before submission, identify the form factor and whether the artifact is a new
app, an update, or an existing listing being made available to new users.
Then use [references/play-target-api-policy.md](references/play-target-api-policy.md)
for the applicable floor, extension date, and exemption. Do not treat an app
that remains usable by previous installers as necessarily discoverable to new
users.
