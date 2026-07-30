# R8, Packaging, and Test Infrastructure

Use this reference for shrinker configuration, resource shrinking, packaging,
shader compilation, fused libraries, reports, and test dashboards. The source
batch is `agp-9-toolchain`.

## Contents

- [Stricter shrinker inputs](#stricter-shrinker-inputs)
- [Kotlin null-check rewriting](#kotlin-null-check-rewriting)
- [Desugaring keep behavior](#desugaring-keep-behavior)
- [Mapping identification](#mapping-identification)
- [Annotation attributes](#annotation-attributes)
- [Negated member names](#negated-member-names)
- [Optimization DSL and source sets](#optimization-dsl-and-source-sets)
- [Configuration analysis](#configuration-analysis)
- [Shader compilation](#shader-compilation)
- [Removed packaging and report features](#removed-packaging-and-report-features)
- [Fused library publication](#fused-library-publication)
- [Aggregated test and coverage dashboards](#aggregated-test-and-coverage-dashboards)

## Stricter shrinker inputs

AGP 9.0 fails the build when a configured keep file is missing. It also
enables optimized resource shrinking and strict full-mode keep semantics.
Keeping a class no longer implicitly retains its default constructor:

```proguard
-keep class A { <init>(); }
```

Library and feature publication rejects global options such as
`-dontoptimize` and `-dontobfuscate` in consumer rules. When precompiled
dependency rules contain these options, app builds silently ignore them.

## Kotlin null-check rewriting

R8 supports `-processkotlinnullchecks` with three values:

- `keep`
- `remove_message`
- `remove`

The default is `remove_message`. If configuration supplies the option more
than once, the strongest value wins.

```proguard
-processkotlinnullchecks keep
```

## Desugaring keep behavior

R8 no longer propagates keep information from interface methods to synthesized
companion methods. This breaks the former separately desugared `minSdk < 24`
library flow that relied on `-applymapping`. Explicitly keep the companion
methods in that separately desugared artifact.

Direct D8/R8 users must replace the removed L8 keep-rule generation APIs and
`--desugared-lib-pg-conf-output` with `TraceReferences`.
`-addconfigurationdebugging` is no longer supported.

## Mapping identification

When retracing is required, R8 writes `r8-map-id-<MAP_ID>` into `SourceFile`
instead of a source filename. The ID is the full mapping hash. A custom
`-renamesourcefileattribute` takes precedence.

In ProGuard compatibility mode, do not keep `SourceFile` if this mapping ID is
needed; retaining the attribute prevents the marker from being emitted.

## Annotation attributes

In AGP 9.2, wildcard `-keepattributes` patterns no longer match
runtime-invisible annotation attributes. Preserve them by naming all three:

```proguard
-keepattributes RuntimeInvisibleAnnotations,
                RuntimeInvisibleParameterAnnotations,
                RuntimeInvisibleTypeAnnotations
```

## Negated member names

AGP 9.2 accepts negated member-name patterns, including in `-if`
preconditions:

```proguard
-keepclassmembers class com.example.MyClass { *** !*ForTesting(...); }
```

Wildcards inside a negated precondition cannot be back-referenced in its
consequent.

## Optimization DSL and source sets

AGP 9.3 adds an `optimization` block to application build types. Enabling it
turns on code optimization and optimized resource shrinking without requiring
the default Android keep-rules file. The legacy DSL remains supported.

`.keep` files in `src/<variant>/keepRules/` work with either DSL for
applications and libraries. They can also define KMP consumer rules.

## Configuration analysis

AGP 9.3 can analyze R8 configuration without finishing an APK or app bundle:

```shell
./gradlew :app:analyzeReleaseR8Config
```

The task produces an R8 Configuration Analyzer report.

## Shader compilation

When shader compilation is enabled, AGP 9.0 requires an explicit compiler path
in `local.properties`:

```properties
glslc.dir=/path/to/shader-tools
```

During migration, opting out of `android.custom.shader.path.required`
temporarily restores the former implicit lookup.

## Removed packaging and report features

AGP 9.0 removes:

- Embedded Wear OS apps and the `wearApp` configuration. Publish the Wear app
  separately.
- Density-split APKs. Use app bundles for density delivery.
- The `androidDependencies` and `sourceSets` report tasks.

## Fused library publication

The preview Fused Library Plugin can combine several Android libraries and
publish them as one Android Library AAR.

## Aggregated test and coverage dashboards

AGP 9.2 has experimental HTML dashboards that aggregate unit and
instrumentation results and coverage across modules and variants. Enable them
with:

```properties
android.experimental.reportAggregationSupport=true
```
