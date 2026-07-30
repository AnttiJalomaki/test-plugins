# Setup, Runtime, and State

## Configure the Android and Kotlin toolchain

- Starting with Compose 1.12.0, Android projects require Android Gradle Plugin
  9 and `compileSdk = 37`. `compileSdk` remains independent of `targetSdk`, so
  this does not itself require targeting SDK 37 (`compiler-toolchain`).
- Compose Animation, Foundation, Runtime, and UI require API 23 or newer from
  1.10.0. Consuming artifacts built with Kotlin 2.0 requires Kotlin Gradle
  Plugin 2.0.0 or newer.
- Compose lint checks require AGP 8.8.2 or newer and Android Studio Ladybug or
  newer from 1.9.0. An older AGP can select standalone Lint 8.8.2 or newer with
  `android.experimental.lint.version=8.8.2` in `gradle.properties`.
- Compose UI's JSpecify type-use nullness can be enforced with
  `-Xjspecify-annotations=strict`; Kotlin 2.1.0 already makes this the default
  (1.8.0).

```kotlin
android {
    compileSdk = 37
}
```

## Select and import a BOM

The ordinary Compose BOM setup uses
`androidx.compose:compose-bom:2026.06.00`. Import the platform for both
application and instrumented-test dependencies, and leave versions off
individual Compose libraries (`compiler-toolchain`).

```kotlin
dependencies {
    val composeBom = platform("androidx.compose:compose-bom:2026.06.00")
    implementation(composeBom)
    androidTestImplementation(composeBom)
    implementation("androidx.compose.material3:material3")
}
```

For testing prereleases, `compose-bom-alpha` selects each library's latest
alpha, beta, RC, or stable release. `compose-bom-beta` selects each library's
latest beta, RC, or stable release. These BOMs can therefore contain a mixture
of stable and prerelease artifacts (`bom-versioning`).

```kotlin
dependencies {
    val composeBom =
        platform("androidx.compose:compose-bom-beta:2026.06.00")
    implementation(composeBom)
}
```

## Configure compiler output

The module-level `composeCompiler` block can put reports under the module build
directory and read a stability configuration from the root project
(`compiler-toolchain`).

```kotlin
composeCompiler {
    reportsDestination = layout.buildDirectory.dir("compose_compiler")
    stabilityConfigurationFile =
        rootProject.layout.projectDirectory.file("stability_config.conf")
}
```

`PausableComposition` requires corresponding compiler support. It can pause a
subcomposition during composition and apply it asynchronously (1.8.0).

## Use runtime annotations without the runtime

The `runtime-annotation` library lets a non-Compose module use `@Stable`,
`@Immutable`, and `@StableMarker` without depending on Compose Runtime. It also
provides `@FrequentlyChangingValue`, with lint for direct composition reads,
and `@RememberInComposition`, with lint for unremembered construction or calls
inside composition (1.9.0).

`PagerState.currentPageOffsetFraction` and `ScrollState.value` are annotated
`@FrequentlyChangingValue` from 1.10.0. Avoid direct composition reads when
they cause avoidable high-frequency invalidation.

## Save state positionally

- Android `SnapshotStateList` and `SnapshotStateSet` are `Parcelable` and can
  be stored with `rememberSaveable` (1.9.0).
- Use `rememberSerializable` for the overload backed by `KSerializer`.
  `rememberSaveable` remains the name for `Saver`-based state (1.9.0).
- Remove the deprecated custom `key` from `rememberSaveable`. It bypasses
  positional scoping and can share or lose state, especially in nested lazy
  layouts (1.9.0).
- Import `LocalSavedStateRegistryOwner` from `androidx.savedstate.compose`.
  `SaveableStateHolder.SaveableStateProvider` supplies that owner to its
  content (1.9.0).
- `StateRestorationTester` always applies platform-specific state encoding
  from 1.10.0.

## Retain values without serializing them

`retain` keeps a value after its composable leaves the hierarchy, but has a
shorter lifetime than saveable state. Android's lifecycle-aware retain scope
also crosses configuration changes. Keys passed to `retain` are themselves
retained, so avoid keys that can leak resources and annotate unsuitable types
with `@DoNotRetain` (1.10.0).

`RetainedEffect` follows the retention lifecycle rather than composition.
`RetainObserver.onUnused` corresponds to `RememberObserver.onAbandoned`.
Custom stores implement `RetainedValuesStore`, can use
`ManagedRetainedValuesStore`, and must be installed with
`LocalRetainedValuesStoreProvider` rather than directly providing
`LocalRetainedValuesStore` (1.10.0).

```kotlin
val store = retainManagedRetainedValuesStore()
LocalRetainedValuesStoreProvider(store) { content() }
```

`RetainedValuesStore.getExitedValueOrDefault` was renamed to
`consumeExitedValueOrDefault`. The experimental concurrent-recomposition API
was removed, and tooling can inspect experimental `RecomposerInfo.errorState`
(1.11.0).

## Handle pausable-composition lifecycle

`PausableComposition` exposes `isApplied` and `isCancelled`. Dispose a
cancelled instance; trying to reuse it throws (1.9.0).

## Work with snapshots and composition keys

- Use `Snapshot.snapshotId` instead of deprecated `Snapshot.id`. The widened
  identifier avoids `Int` overflow. `SnapshotId` arithmetic and special
  constants are internal; convert with `toInt()` or `toLong()` only when
  arithmetic is required (1.8.0).
- `currentCompositeKeyHash` is deprecated in favor of
  `currentCompositeKeyHashCode`, which carries more bits and reduces unrelated
  composition-group collisions (1.9.0).

## Observe composition completion

`awaitOrScheduleNextCompositionEnd()` invokes its callback after the current
frame's composition, or schedules and waits for the next frame when the
recomposer is idle. Composition-local providers can return a non-`Unit` value,
and composition-registration observers run before initial composition
(1.10.0).

## Diagnose Compose stacks

`setDiagnosticStackTraceEnabled` is experimental. Compose stack traces also
cover work launched by `LaunchedEffect` and `rememberCoroutineScope` (1.9.0).

`ComposeStackTraceMode.GroupKeys` supports minified applications. It is off by
default; Kotlin 2.3.0's Compose compiler Gradle plugin begins generating the
required mapping (1.10.0).

## Use the multiplatform runtime artifacts precisely

- `androidx.compose.runtime:runtime` publishes desktop, iOS, and native support
  through Google Maven, but that expansion applies only to Runtime, not all of
  AndroidX Compose (1.9.0).
- `runtime-rxjava2` and `runtime-rxjava3` are multiplatform and include JVM
  targets (1.10.0).

## Delete removed runtime flags

Remove assignments to
`isMovingNestedMovableContentEnabled` and
`isMovableContentUsageTrackingEnabled`; these runtime flags no longer exist
(1.11.0).
