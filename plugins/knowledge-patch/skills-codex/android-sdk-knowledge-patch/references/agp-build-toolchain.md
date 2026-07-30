# Android Gradle Plugin and Build Toolchain

## Select a compatible toolchain

The `agp-9-toolchain` guidance has these floors and defaults:

| AGP | Maximum supported SDK | Gradle | JDK | Build Tools | Default NDK |
|---|---:|---:|---:|---:|---:|
| 9.0 | API 36.1 | 9.1.0 | 17 | 36.0.0 | 28.2.13676358 (`r28c`) |
| 9.2 | API 37.0 | 9.4.1 | 17 | 36.0.0 | 28.2.13676358 (`r28c`) |
| 9.3 | API 37 | 9.5.0 | 17 | 36.0.0 | 28.2.13676358 (`r28c`) |

AGP 9.0 also makes a library's compile SDK the default minimum compile SDK for consumers unless its publisher explicitly sets `AarMetadata.minCompileSdk`.

## Migrate build logic to public APIs

AGP 9.0 makes the public DSL and Variant API the default and hides legacy DSL implementations and variant entry points.

- Replace `applicationVariants` and sibling collections with `androidComponents.onVariants`.
- Replace `variantFilter` with `androidComponents.beforeVariants`.
- Obtain SDK paths from `androidComponents.sdkComponents`.
- Replace custom test providers with Gradle-managed devices.

```kotlin
androidComponents {
    beforeVariants(selector().withBuildType("debug")) { it.enable = false }
}
```

`android.newDsl=false` temporarily restores the legacy implementation for incompatible plugins, but AGP 10 removes the escape hatch.

### Plugin API replacements

- Move bytecode transformation and ASM-frame configuration from `Component` to `Instrumentation`.
- Replace `ComponentBuilder.enabled` with `enable`.
- Replace `VariantOutput.enable` with `enabled`.
- Replace variant `*SdkVersion` properties with `minSdk`, `maxSdk`, or `targetSdk`.
- Access unit-test members only through the relevant `HasUnitTest` or `HasUnitTestBuilder` subtype.
- Replace removed `BaseExtension.registerTransform` integrations.

### DSL type changes

`CommonExtension` is no longer parameterized. Invoke its block methods on a concrete extension or through properties such as `defaultConfig.apply`.

- Replace `DependencyVariantSelection` with `DependencySelection` at `kotlin.android.localDependencySelection`.
- Replace `Installation.installOptions(String)` with the mutable `installOptions` property.
- Replace `ProductFlavor.setDimension` with `dimension`.
- Remove uses of deleted `DensitySplit`, `LanguageSplitOptions`, and experimental `PostProcessing` APIs.

## Adopt built-in Kotlin and KMP integration

AGP 9.0 enables built-in Kotlin. Android modules must stop applying `org.jetbrains.kotlin.android` or `kotlin-android`. AGP carries KGP 2.2.10 and upgrades lower KGP versions; it similarly raises KSP versions below 2.2.10-2.0.2.

Supply a higher KGP version as a top-level `buildscript` classpath dependency. A strictly constrained lower KGP, with a minimum of 2.0.0, is possible only after opting out of built-in Kotlin.

The new DSL cannot combine `org.jetbrains.kotlin.multiplatform` with `com.android.application` or `com.android.library` in one subproject. Use the Android Gradle Library Plugin for KMP. Move an Android application to a separate subproject because the new KMP integration does not support the application plugin in the KMP module.

## Review changed Android defaults

AGP 9.0 changes these defaults:

- library package names must be unique;
- AndroidX is the default dependency family;
- application code compiles against a non-final `R`;
- an unset target SDK defaults to the compile SDK, not the minimum SDK;
- `resValues` is disabled unless enabled in the module;
- generated-source providers must use the `androidComponents` Sources API instead of `AndroidSourceSet`.

On-device tests now default to `AndroidJUnitRunner`. Only the tested build type gets a unit-test component by default, normally debug rather than both debug and release. `android.dependency.useConstraints` defaults to `false`, limiting dependency constraints to application device tests unless the old behavior is explicitly restored.

Enable AIDL or RenderScript only in modules that need it. The global `android.defaults.buildfeatures.aidl` and `android.defaults.buildfeatures.renderscript` properties are removed.

## Update shrinking and keep rules

AGP 9.0 fails the build for missing keep files, enables optimized resource shrinking, and applies strict full-mode keep semantics. `-keep class A` no longer retains the default constructor; name it:

```proguard
-keep class A { <init>(); }
```

Library and feature publication rejects global consumer-rule options such as `-dontoptimize` and `-dontobfuscate`. When precompiled dependency rules contain those options, an app build silently ignores them.

The properties `android.r8.integratedResourceShrinking` and `android.enableNewResourceShrinker.preciseShrinking` are errors because integrated, precise resource shrinking is mandatory.

### Kotlin null checks

R8's `-processkotlinnullchecks` accepts `keep`, `remove_message`, or `remove`; the default is `remove_message`, and the strongest value wins when the option appears more than once.

```proguard
-processkotlinnullchecks keep
```

### Desugaring keep rules

Keep information on interface methods is no longer propagated to synthesized companion methods. For a separately desugared `minSdk < 24` library that previously depended on `-applymapping`, explicitly keep the companion methods.

Direct D8/R8 integrations must replace the removed L8 keep-rule generation APIs and `--desugared-lib-pg-conf-output` with `TraceReferences`. `-addconfigurationdebugging` is no longer supported.

### Mapping IDs

R8 writes `r8-map-id-<MAP_ID>` with the full mapping hash into `SourceFile` for obfuscated code. A custom `-renamesourcefileattribute` takes precedence. In ProGuard compatibility mode, do not keep `SourceFile` if the mapping ID is required.

### Runtime-invisible annotations

In AGP 9.2, wildcard `-keepattributes` patterns no longer select runtime-invisible annotation attributes. Preserve all three explicitly when required:

```proguard
-keepattributes RuntimeInvisibleAnnotations,
                RuntimeInvisibleParameterAnnotations,
                RuntimeInvisibleTypeAnnotations
```

### Negated members

AGP 9.2 accepts negated member-name patterns, including in `-if` preconditions. Wildcards inside a negated precondition cannot be back-referenced in its consequent.

```proguard
-keepclassmembers class com.example.MyClass { *** !*ForTesting(...); }
```

## Handle removed packaging and reports

AGP 9.0 removes embedded Wear OS apps and the `wearApp` configuration, density-split APKs, and the `androidDependencies` and `sourceSets` report tasks. Publish Wear apps separately and use app bundles for density delivery.

When shader compilation is enabled, set `glslc.dir=/path/to/shader-tools` in `local.properties`. The old implicit lookup is available during migration only by opting out of `android.custom.shader.path.required`.

## Use newer build features

The preview Fused Library Plugin can combine several Android libraries into one Android Library AAR.

With AGP 9.2, `android.experimental.reportAggregationSupport=true` enables experimental HTML dashboards aggregating unit tests, instrumentation tests, and coverage across modules and variants.

With AGP 9.3, run `./gradlew :app:analyzeReleaseR8Config` to generate the R8 Configuration Analyzer report without completing an APK or app bundle build.

AGP 9.3 adds an `optimization` block to app build types. Enabling it turns on code optimization and optimized resource shrinking without the default Android keep-rules file; the legacy DSL remains supported. `.keep` files under `src/<variant>/keepRules/` work with either DSL for apps and libraries and can define KMP consumer rules.

## Prepare plugins for AGP 10

The AGP 10 lazy build architecture removes `android.newDsl`, `android.builtInKotlin`, every legacy extension and Variant API, direct task access, eager generated-source registration, and the Transform API.

Custom build logic must use `Variant.artifacts`, `variant.sources.*.addGeneratedSourceDirectory`, `variant.instrumentation.transformClassesWith`, and lazy properties. Compile plugins against `gradle-api`; the `gradle` artifact will no longer expose internal AGP classes.

Stage the behavior on AGP 9.x:

```properties
android.newDsl=true
android.builtInKotlin=true
```

Starting in AGP 9.4.0-alpha04, `android.newDsl.optOut=:lib` can temporarily exempt named modules, but AGP 10 removes it. A module with no Kotlin can use `android { enableKotlin = false }` to omit its Kotlin compiler task and standard-library dependency.
