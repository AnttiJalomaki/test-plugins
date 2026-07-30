# AGP Toolchain and Public APIs

Use this reference when selecting an Android Gradle Plugin toolchain,
migrating build logic to public lazy APIs, configuring Kotlin, or preparing a
plugin for the AGP 10 build model. These changes derive from the
`agp-9-toolchain` batch.

## Contents

- [Supported toolchain combinations](#supported-toolchain-combinations)
- [Public DSL and Variant API migration](#public-dsl-and-variant-api-migration)
- [Kotlin and KMP integration](#kotlin-and-kmp-integration)
- [Changed Android defaults](#changed-android-defaults)
- [DSL shape changes](#dsl-shape-changes)
- [Feature flags and module configuration](#feature-flags-and-module-configuration)
- [Preparing for AGP 10](#preparing-for-agp-10)

## Supported toolchain combinations

| AGP | Supported Android API | Required Gradle | JDK | Build Tools | Default NDK |
| --- | --- | --- | --- | --- | --- |
| 9.0 | Through API 36.1 | 9.1.0 | 17 | 36.0.0 | 28.2.13676358 (`r28c`) |
| 9.2 | API 37.0 | 9.4.1 | 17 | 36.0.0 | 28.2.13676358 (`r28c`) |
| 9.3 | API 37 | 9.5.0 | 17 | 36.0.0 | 28.2.13676358 (`r28c`) |

In AGP 9.0, a library's compile SDK becomes the default minimum compile SDK
for consumers. Publishers that require another floor must set
`AarMetadata.minCompileSdk` explicitly.

## Public DSL and Variant API migration

AGP 9.0 hides legacy DSL implementations and legacy variant entry points. Use
the public DSL, `androidComponents`, lazy properties, and artifact providers.

| Removed or legacy API | Replacement |
| --- | --- |
| `applicationVariants` and sibling collections | `androidComponents.onVariants` |
| `variantFilter` | `androidComponents.beforeVariants` |
| SDK path getters | `androidComponents.sdkComponents` |
| Custom test providers | Gradle-managed devices |
| `Component` bytecode transforms | `Instrumentation` |
| `Component` ASM frame configuration | `Instrumentation` |
| `ComponentBuilder.enabled` | `enable` |
| `VariantOutput.enable` | `enabled` |
| `*SdkVersion` variant properties | `minSdk`, `maxSdk`, or `targetSdk` |
| `BaseExtension.registerTransform` | Instrumentation transform APIs |

Example variant disabling:

```kotlin
androidComponents {
    beforeVariants(selector().withBuildType("debug")) { it.enable = false }
}
```

Unit-test members are available only on the relevant `HasUnitTest` and
`HasUnitTestBuilder` subtypes. Do not cast every component to an implementation
type merely to recover removed members.

`android.newDsl=false` temporarily exposes the old implementation to
incompatible plugins on AGP 9. The switch is removed in AGP 10.

## Kotlin and KMP integration

### Built-in Kotlin

AGP 9.0 enables built-in Kotlin. Android modules must stop applying
`org.jetbrains.kotlin.android` or `kotlin-android` in parallel.

AGP carries Kotlin Gradle Plugin 2.2.10. It upgrades lower KGP versions to
2.2.10 and KSP versions below 2.2.10-2.0.2 to that KSP version. Supply a higher
KGP as a top-level `buildscript` classpath dependency. A lower strictly
constrained KGP is supported only down to 2.0.0 and only after opting out of
built-in Kotlin.

### Kotlin Multiplatform

The new DSL cannot combine `org.jetbrains.kotlin.multiplatform` with
`com.android.application` or `com.android.library` in one subproject. Use the
Android Gradle Library Plugin for the KMP module. Put an Android application in
a separate subproject because the new KMP integration does not support the
application plugin inside the KMP module.

### Local dependency selection

Replace `DependencyVariantSelection` with `DependencySelection`, configured at
`kotlin.android.localDependencySelection`.

## Changed Android defaults

AGP 9.0 changes these defaults and requirements:

- Library package names must be unique.
- AndroidX is the default dependency family.
- Application code compiles against a non-final `R`.
- If `targetSdk` is unset, it defaults to `compileSdk`, not `minSdk`.
- `resValues` is disabled until enabled in each module that needs it.
- Generated-source providers must be registered with the
  `androidComponents` Sources API, not through `AndroidSourceSet`.
- On-device tests use `AndroidJUnitRunner` by default.
- Only the tested build type receives a unit-test component by default,
  normally debug rather than both debug and release.
- `android.dependency.useConstraints` defaults to `false`, which limits
  dependency constraints to application device tests unless old behavior is
  restored explicitly.

## DSL shape changes

- `CommonExtension` is no longer parameterized.
- Invoke its block methods on a concrete extension or through properties such
  as `defaultConfig.apply`.
- Replace `Installation.installOptions(String)` with the mutable
  `installOptions` property.
- Replace `ProductFlavor.setDimension` with the `dimension` property.
- `DensitySplit`, `LanguageSplitOptions`, and the experimental
  `PostProcessing` block are removed.

## Feature flags and module configuration

The global `android.defaults.buildfeatures.aidl` and
`android.defaults.buildfeatures.renderscript` properties are removed. Enable
`aidl` or `renderScript` only in modules that use the feature.

AGP 9.0 rejects `android.r8.integratedResourceShrinking` and
`android.enableNewResourceShrinker.preciseShrinking`. Integrated precise
resource shrinking is mandatory; remove both properties.

## Preparing for AGP 10

AGP 10's planned lazy build model removes:

- `android.newDsl` and `android.builtInKotlin`.
- Every legacy extension and Variant API.
- Direct task access and eager generated-source registration.
- The Transform API.

Custom build logic must use `Variant.artifacts`,
`variant.sources.*.addGeneratedSourceDirectory`,
`variant.instrumentation.transformClassesWith`, and lazy properties. Plugin
projects must compile against `gradle-api`; the `gradle` artifact will no
longer expose internal AGP classes.

Stage strict behavior on AGP 9.x with:

```properties
android.newDsl=true
android.builtInKotlin=true
```

Starting in AGP 9.4.0-alpha04, `android.newDsl.optOut=:lib` can temporarily
exempt named modules. That escape hatch disappears in AGP 10. A module with no
Kotlin can set `android { enableKotlin = false }` to avoid its Kotlin compiler
task and standard-library dependency.
