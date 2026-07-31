# Build, Runtime, and State

## Build and dependency setup

### Compose lint toolchain floor (1.9.0)

Compose lint checks require AGP 8.8.2 or newer and Android Studio Ladybug or
newer. If the project must stay on an older AGP, select standalone Lint 8.8.2
or newer:

```properties
android.experimental.lint.version=8.8.2
```

### Core platform and Kotlin floor (1.10.0)

Compose Animation, Foundation, Runtime, and UI have a minimum SDK of API 23.
Artifacts built with Kotlin 2.0 require Kotlin Gradle Plugin 2.0.0 or newer in
the consuming build.

### Android toolchain for Compose 1.12 (compiler-toolchain)

Starting with Compose 1.12.0, Android projects must use `compileSdk = 37` and
Android Gradle Plugin 9. The compile SDK is independent of `targetSdk`; adopting
the build requirement does not itself require targeting API 37.

```kotlin
android {
    compileSdk = 37
}
```

### Compiler reports and stability configuration (compiler-toolchain)

Configure reports and a root-project stability file in the module-level
`composeCompiler` block:

```kotlin
composeCompiler {
    reportsDestination = layout.buildDirectory.dir("compose_compiler")
    stabilityConfigurationFile =
        rootProject.layout.projectDirectory.file("stability_config.conf")
}
```

### Compose BOM 2026.06.00 (compiler-toolchain)

Import the platform in application and instrumented-test configurations, then
omit versions from aligned Compose libraries:

```kotlin
dependencies {
    val composeBom = platform("androidx.compose:compose-bom:2026.06.00")
    implementation(composeBom)
    androidTestImplementation(composeBom)
    implementation("androidx.compose.material3:material3")
}
```

### Alpha and beta BOM artifacts (bom-versioning)

`compose-bom-alpha` selects each library's newest alpha, beta, RC, or stable
release. `compose-bom-beta` selects each library's newest beta, RC, or stable
release. They are testing-oriented BOMs and can mix stable and prerelease
libraries; inspect the resolved dependency graph rather than assuming every
artifact is prerelease.

```kotlin
dependencies {
    val composeBom = platform(
        "androidx.compose:compose-bom-beta:2026.06.00"
    )
    implementation(composeBom)
}
```

## Runtime annotations and diagnostics

### JSpecify nullness (1.8.0)

Compose UI includes type-use JSpecify annotations. Kotlin can enforce them with
`-Xjspecify-annotations=strict`; Kotlin 2.1.0 already defaults to strict mode.

### Runtime annotations without the runtime dependency (1.9.0)

Non-Compose modules can depend on `runtime-annotation` to use `@Stable`,
`@Immutable`, and `@StableMarker` without pulling in Compose Runtime. It also
contains:

- `@FrequentlyChangingValue`, whose lint warns about direct composition reads.
- `@RememberInComposition`, whose lint rejects construction or calls in
  composition unless they are remembered.

### Composite-key hashes (1.9.0)

Replace deprecated `currentCompositeKeyHash` with
`currentCompositeKeyHashCode`. The newer value carries more hash bits and
reduces collisions between unrelated composition groups.

### Compose stack traces (1.9.0)

`setDiagnosticStackTraceEnabled` is experimental. Diagnostic Compose stack
traces include work launched by `LaunchedEffect` and `rememberCoroutineScope`.

### Group-key stack traces (1.10.0)

`ComposeStackTraceMode.GroupKeys` supports useful Compose traces in minified
apps. It is off by default. Kotlin 2.3.0's Compose compiler Gradle plugin starts
generating the required group-key mapping.

### Recomposer tooling and concurrent recomposition (1.11.0)

The experimental concurrent-recomposition API was removed. Tooling can inspect
the experimental `RecomposerInfo.errorState` instead.

## Snapshots and composition lifecycle

### Snapshot identifiers (1.8.0)

Use `Snapshot.snapshotId` instead of deprecated `Snapshot.id`. The widened ID
avoids `Int` overflow in long-running, high-frame-rate processes. Arithmetic
and special `SnapshotId` constants are internal; convert with `toInt()` or
`toLong()` only when arithmetic is unavoidable.

### Pausable composition and compiler support (1.8.0)

`PausableComposition` can pause a subcomposition while it is composed and
apply the result asynchronously. It requires corresponding Compose compiler
support.

### Pausable-composition lifecycle (1.9.0)

Check `isApplied` and `isCancelled`. A cancelled pausable composition must be
disposed; reusing it throws.

### Runtime completion hooks (1.10.0)

`awaitOrScheduleNextCompositionEnd()` invokes its callback after the current
frame's composition, or schedules and awaits another frame when the recomposer
is idle. Composition-local providers may return non-`Unit` values, and
composition-registration observers run before initial composition.

## Saveable state

### Saveable collections and serialization (1.9.0)

On Android, `SnapshotStateList` and `SnapshotStateSet` implement `Parcelable`
and can be stored by `rememberSaveable`. Use `rememberSerializable` for the
`KSerializer` overload. The `Saver`-based API keeps the
`rememberSaveable` name.

### Positional state and registry owners (1.9.0)

The custom-`key` overload of `rememberSaveable` is deprecated because it
bypasses positional scoping and can share or lose state, notably in nested lazy
layouts. Remove the key. Import `LocalSavedStateRegistryOwner` from
`androidx.savedstate.compose`; `SaveableStateHolder.SaveableStateProvider`
supplies that owner to its content.

## Retained state

### Retained values (1.10.0)

`retain` preserves a value after its composable leaves the hierarchy without
serializing it. Its lifetime is shorter than saveable state. Android's
lifecycle-aware retain scope can carry retained values across configuration
changes. Keys are retained as well, so do not use keys that hold resources or
other leak-prone objects; annotate unsuitable types with `@DoNotRetain`.

### Effects and custom retained stores (1.10.0)

`RetainedEffect` follows the retention lifecycle rather than the composition
lifecycle. `RetainObserver.onUnused` is the retention counterpart of
`RememberObserver.onAbandoned`.

Custom stores implement `RetainedValuesStore`; use
`ManagedRetainedValuesStore` and install the store with
`LocalRetainedValuesStoreProvider`, not by directly providing
`LocalRetainedValuesStore`:

```kotlin
val store = retainManagedRetainedValuesStore()
LocalRetainedValuesStoreProvider(store) { content() }
```

### Retention API rename (1.11.0)

Replace `RetainedValuesStore.getExitedValueOrDefault` with
`consumeExitedValueOrDefault`; consuming makes the operation's ownership
semantics explicit.

## Multiplatform runtime artifacts

### Runtime from Google Maven (1.9.0)

`androidx.compose.runtime:runtime` publishes desktop, iOS, and native variants
through Google Maven, upstreamed from Compose Multiplatform. This applies to
Runtime artifacts only, not all AndroidX Compose libraries.

### RxJava runtime targets (1.10.0)

`runtime-rxjava2` and `runtime-rxjava3` are multiplatform and include JVM as a
supported target.
