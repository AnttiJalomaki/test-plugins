# Build, Runtime, and State

## Android, Kotlin, and lint requirements

- Compose lint checks require Android Gradle Plugin 8.8.2 or newer and Android
  Studio Ladybug or newer (since 1.9.0). When the project must remain on an
  older AGP, select standalone Lint 8.8.2 or newer in `gradle.properties`:

  ```properties
  android.experimental.lint.version=8.8.2
  ```

- Compose Animation, Foundation, Runtime, and UI raise `minSdk` from API 21 to
  API 23 in 1.10.0. Artifacts built with Kotlin 2.0 require Kotlin Gradle
  Plugin 2.0.0 or newer in the consuming build.
- Starting with Compose 1.12.0, Android projects need `compileSdk = 37` and
  Android Gradle Plugin 9. `compileSdk` and `targetSdk` remain independent, so
  this requirement does not by itself require targeting SDK 37.

  ```kotlin
  android {
      compileSdk = 37
  }
  ```

## Compiler reports and stability

Configure compiler reports and the root-project stability file in the
module-level `composeCompiler` block:

```kotlin
composeCompiler {
    reportsDestination = layout.buildDirectory.dir("compose_compiler")
    stabilityConfigurationFile =
        rootProject.layout.projectDirectory.file("stability_config.conf")
}
```

`PausableComposition` also needs corresponding compiler support. For minified
stack traces, `ComposeStackTraceMode.GroupKeys` requires group-key mappings;
the Compose compiler Gradle plugin starts generating them with Kotlin 2.3.0.

## BOM selection

The standard setup described by the compiler-toolchain batch uses
`androidx.compose:compose-bom:2026.06.00`. Import the platform for application
and instrumented-test dependencies, then omit individual library versions:

```kotlin
dependencies {
    val composeBom = platform("androidx.compose:compose-bom:2026.06.00")
    implementation(composeBom)
    androidTestImplementation(composeBom)
    implementation("androidx.compose.material3:material3")
}
```

The bom-versioning guidance adds testing-only prerelease BOMs. Use
`compose-bom-alpha` to select each library's latest alpha, beta, RC, or stable
release, or `compose-bom-beta` to select its latest beta, RC, or stable
release. Either BOM can therefore resolve some libraries to stable versions.

```kotlin
dependencies {
    val composeBom =
        platform("androidx.compose:compose-bom-beta:2026.06.00")
    implementation(composeBom)
}
```

## Nullness and runtime annotations

Compose UI carries type-use JSpecify nullness annotations (since 1.8.0).
Kotlin builds can enforce them with `-Xjspecify-annotations=strict`; Kotlin
2.1.0 already makes strict handling the default.

The `runtime-annotation` library lets a non-Compose module use `@Stable`,
`@Immutable`, and `@StableMarker` without a Compose Runtime dependency (since
1.9.0). It also supplies:

- `@FrequentlyChangingValue`, whose lint check warns when such a value is read
  directly during composition.
- `@RememberInComposition`, whose lint check rejects construction or calls in
  composition that were not remembered.

`PagerState.currentPageOffsetFraction` and `ScrollState.value` carry
`@FrequentlyChangingValue` as of 1.10.0. Avoid direct composition reads when
they cause unnecessary recomposition; derive or collect the value at the
appropriate rate instead.

## Snapshot and composition identity

- Use `Snapshot.snapshotId`, not deprecated `Snapshot.id` (since 1.8.0). The
  wider identifier avoids `Int` overflow in long-running, high-frame-rate
  processes. `SnapshotId` arithmetic and special constants are internal; use
  `toInt()` or `toLong()` only when arithmetic is unavoidable.
- Use `currentCompositeKeyHashCode`, not deprecated
  `currentCompositeKeyHash` (since 1.9.0). It preserves more hash bits and
  reduces collisions between unrelated composition groups.

## Saveable state and serialization

On Android, `SnapshotStateList` and `SnapshotStateSet` are `Parcelable` and
can be stored by `rememberSaveable` (since 1.9.0). The `KSerializer` overload
is called `rememberSerializable`; the `Saver`-based `rememberSaveable` remains
available.

Remove the deprecated custom `key` argument from `rememberSaveable`. It
bypasses positional scoping and can share or lose state, particularly in
nested lazy layouts. Import `LocalSavedStateRegistryOwner` from
`androidx.savedstate.compose`. `SaveableStateHolder.SaveableStateProvider`
supplies that owner to its content.

## Pausable composition

`PausableComposition` can pause a subcomposition during composition and apply
it asynchronously (since 1.8.0). It exposes `isApplied` and `isCancelled`
(since 1.9.0). A cancelled instance must be disposed; reuse now throws.

## Retained values and effects

`retain` keeps a value after its composable leaves the hierarchy without
serializing it (since 1.10.0). Its lifetime is shorter than saveable state;
Android's lifecycle-aware retain scope also carries retained values across
configuration changes. Keys are retained with their values, so avoid keys
that can leak resources. Mark unsuitable types with `@DoNotRetain`.

`RetainedEffect` follows the retention lifecycle rather than the composition
lifecycle. `RetainObserver.onUnused` is the retention analogue of
`RememberObserver.onAbandoned`.

For custom storage, implement `RetainedValuesStore`, create a
`ManagedRetainedValuesStore`, and install it through
`LocalRetainedValuesStoreProvider`; do not directly provide
`LocalRetainedValuesStore`:

```kotlin
val store = retainManagedRetainedValuesStore()
LocalRetainedValuesStoreProvider(store) { content() }
```

In 1.11.0, `RetainedValuesStore.getExitedValueOrDefault` is renamed to
`consumeExitedValueOrDefault`. The experimental concurrent-recomposition API
is removed. The runtime flags `isMovingNestedMovableContentEnabled` and
`isMovableContentUsageTrackingEnabled` are also removed; delete assignments
because the associated behavior can no longer be selected this way.

## Runtime completion and diagnostics

- `awaitOrScheduleNextCompositionEnd()` invokes its callback after the current
  frame's composition, or schedules and waits for the next frame if the
  recomposer is idle (since 1.10.0).
- Composition-local providers can return a non-`Unit` value, and
  composition-registration observers now run before initial composition.
- `setDiagnosticStackTraceEnabled` is experimental as of 1.9.0. Compose stack
  traces cover work launched by `LaunchedEffect` and `rememberCoroutineScope`.
- `ComposeStackTraceMode.GroupKeys` supports minified applications but is
  disabled by default.
- Tooling can inspect experimental `RecomposerInfo.errorState` (since 1.11.0).

## Multiplatform runtime artifacts

`androidx.compose.runtime:runtime` publishes desktop, iOS, and native variants
through Google Maven (since 1.9.0), upstreamed from Compose Multiplatform.
Only Runtime artifacts receive this expansion, not all AndroidX Compose
libraries. In 1.10.0, `runtime-rxjava2` and `runtime-rxjava3` also become
multiplatform and add JVM as a supported target.
