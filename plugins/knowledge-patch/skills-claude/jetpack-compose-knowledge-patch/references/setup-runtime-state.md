# Setup, Runtime, and State

## Android and compiler toolchains

### Android platform floors

Compose Animation, Foundation, Runtime, and UI 1.10.0 raise `minSdk` from 21
to 23. Artifacts built with Kotlin 2.0 require Kotlin Gradle Plugin 2.0.0 or
newer in consuming builds.

Starting with Compose 1.12, Android projects need `compileSdk = 37` and
Android Gradle Plugin 9. `compileSdk` is independent of `targetSdk`, so this
does not require targeting API 37. Treat this as the `compileSdk 37` floor.

```kotlin
android {
    compileSdk = 37
}
```

### Lint floor (1.9.0)

Compose lint checks require AGP 8.8.2 or newer and Android Studio Ladybug or
newer. If the build must retain an older AGP, select standalone Lint 8.8.2 or
newer in `gradle.properties`:

```properties
android.experimental.lint.version=8.8.2
```

### Compiler reports and stability configuration

Configure reports and the root-project stability file in the module-level
compiler extension:

```kotlin
composeCompiler {
    reportsDestination = layout.buildDirectory.dir("compose_compiler")
    stabilityConfigurationFile =
        rootProject.layout.projectDirectory.file("stability_config.conf")
}
```

### JSpecify nullness (1.8.0)

Compose UI has type-use JSpecify nullness annotations. Kotlin can enforce them
with `-Xjspecify-annotations=strict`; Kotlin 2.1.0 already makes strict mode
the default.

## BOM selection

### Stable BOM

The setup stream uses `androidx.compose:compose-bom:2026.06.00`. Import the
platform in both application and instrumented-test configurations, then omit
versions on individual Compose artifacts:

```kotlin
dependencies {
    val composeBom = platform("androidx.compose:compose-bom:2026.06.00")
    implementation(composeBom)
    androidTestImplementation(composeBom)
    implementation("androidx.compose.material3:material3")
}
```

### Alpha and beta BOMs

`compose-bom-alpha` selects each library's latest alpha, beta, RC, or stable
release. `compose-bom-beta` selects each library's latest beta, RC, or stable
release. Both are testing-oriented and may contain a mixture of stable and
prerelease libraries.

```kotlin
dependencies {
    val composeBom =
        platform("androidx.compose:compose-bom-beta:2026.06.00")
    implementation(composeBom)
}
```

## Runtime annotations and artifacts

### Annotations without the runtime dependency (1.9.0)

Non-Compose modules can depend on `runtime-annotation` to use `@Stable`,
`@Immutable`, and `@StableMarker` without the Compose runtime. It also defines
`@FrequentlyChangingValue`, whose lint warns about direct composition reads,
and `@RememberInComposition`, whose lint rejects construction or calls in
composition that were not remembered.

### Multiplatform artifacts

As of 1.9.0, `androidx.compose.runtime:runtime` publishes desktop, iOS, and
native targets through Google Maven. This upstreamed multiplatform support
applies to runtime artifacts, not all AndroidX Compose libraries.

In 1.10.0, `runtime-rxjava2` and `runtime-rxjava3` also become multiplatform
and add JVM targets.

## Saveable state

### Collections and serialization (1.9.0)

On Android, `SnapshotStateList` and `SnapshotStateSet` implement `Parcelable`
and can be stored by `rememberSaveable`. Use `rememberSerializable` for the
`KSerializer`-based overload; the `Saver`-based API remains named
`rememberSaveable`.

### Positional scoping and registry ownership (1.9.0)

Remove the deprecated custom `key` from `rememberSaveable`. It bypasses
positional scoping and can share or lose state, especially inside nested lazy
layouts.

Import `LocalSavedStateRegistryOwner` from `androidx.savedstate.compose`.
`SaveableStateHolder.SaveableStateProvider` now supplies that owner to its
content.

## Snapshots and composition identity

### Snapshot identifiers (1.8.0)

Use `Snapshot.snapshotId` instead of deprecated `Snapshot.id`; the wider ID
avoids `Int` overflow in long-running, high-frame-rate processes. Arithmetic
and special `SnapshotId` constants are internal. Convert with `toInt()` or
`toLong()` only when arithmetic is required.

### Composite-key hashes (1.9.0)

Replace `currentCompositeKeyHash` with `currentCompositeKeyHashCode`. The new
value carries more hash bits and reduces unrelated composition-group
collisions.

## Pausable composition

### Compiler support and lifecycle

`PausableComposition` can pause a subcomposition while it is composed and
apply the result asynchronously (1.8.0); it needs matching compiler support.

In 1.9.0, inspect `isApplied` and `isCancelled`. Always dispose a cancelled
pausable composition; attempting to reuse one throws.

## Retention

### Retained values (1.10.0)

`retain` keeps a value when its composable temporarily leaves the hierarchy
without serializing it. Its lifetime is shorter than saveable state. On
Android, the lifecycle-aware retain scope carries values across configuration
changes.

Keys passed to `retain` are retained too. Avoid keys that can leak resources,
and annotate unsuitable types with `@DoNotRetain`.

### Effects and stores (1.10.0)

`RetainedEffect` follows retention rather than composition lifetime.
`RetainObserver.onUnused` corresponds to `RememberObserver.onAbandoned`.

Custom stores implement `RetainedValuesStore` or use
`ManagedRetainedValuesStore`. Install a store through
`LocalRetainedValuesStoreProvider`, not by directly providing
`LocalRetainedValuesStore`:

```kotlin
val store = retainManagedRetainedValuesStore()
LocalRetainedValuesStoreProvider(store) { content() }
```

In 1.11.0, rename `getExitedValueOrDefault` to
`consumeExitedValueOrDefault`.

## Composition hooks and diagnostics

### Completion and registration (1.10.0)

`awaitOrScheduleNextCompositionEnd()` invokes its callback after the current
frame's composition. If the recomposer is idle, it schedules and awaits the
next frame. Composition-local providers may return a non-`Unit` value, and
composition-registration observers run before initial composition.

### Stack traces

In 1.9.0, `setDiagnosticStackTraceEnabled` is experimental, and Compose stack
traces include work launched by `LaunchedEffect` and `rememberCoroutineScope`.

`ComposeStackTraceMode.GroupKeys` in 1.10.0 enables stack traces in minified
apps. It is off by default, and Kotlin 2.3.0's Compose compiler plugin begins
generating the required mapping.

Tooling can inspect experimental `RecomposerInfo.errorState` in 1.11.0. The
experimental concurrent-recomposition API is removed.

### Host-default composition locals (1.11.0)

`compositionLocalWithHostDefaultOf` lets a local obtain its fallback from the
host, such as an Android `View` tag. `HostDefaultKey` is now an interface;
custom hosts can implement values with `HostDefaultProvider` and
`LocalHostDefaultProvider`.

## Removed runtime and behavior flags

Foundation 1.10.0 removes temporary flags for non-composed clickables,
on-scroll callbacks, fling continuation, drag pickup, automatic nested
prefetch, pointer-velocity adjustment, and pointer/nested-scroll interop
fixes. Delete assignments when upgrading.

Foundation and Runtime 1.11.0 additionally remove
`isDetectTapGesturesImmediateCoroutineDispatchEnabled`,
`isNonSuspendingPointerInputInClickableEnabled`,
`isTextFieldDpadNavigationEnabled`,
`isKeepInViewFocusObservationChangeEnabled`,
`isMovingNestedMovableContentEnabled`, and
`isMovableContentUsageTrackingEnabled`. UI removes
`isSemanticAutofillEnabled`. The corresponding autofill, D-pad navigation,
and keep-in-view behaviors are always enabled.
