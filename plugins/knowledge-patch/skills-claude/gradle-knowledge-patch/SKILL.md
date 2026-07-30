---
name: gradle-knowledge-patch
description: Gradle
version: 9.6.1
license: MIT
metadata:
  author: Nevaberry
---


# Gradle Knowledge Patch

Use this skill when changing a Gradle build, plugin, convention plugin, Tooling
API client, TestKit test, publication, or wrapper setup. Start with the
migration notes for an upgrade, then load only the topic references needed for
the task.

Prefer the build's wrapper, manifests, source, and observed behavior over
generic advice. Keep daemon JVM requirements separate from compilation
toolchains, preserve lazy configuration, and test configuration-cache behavior
explicitly when build logic participates in it.

## Reference index

| Reference | Topics |
| --- | --- |
| [migration-gradle-9.md](references/migration-gradle-9.md) | Runtime and language baselines, removed APIs and DSLs, changed task defaults, publication and cache migration |
| [jvm-languages-build-logic.md](references/jvm-languages-build-logic.md) | Daemon and Java toolchains, Scala, Kotlin and Groovy build logic, ANTLR, plugin authoring |
| [testing-quality-problems.md](references/testing-quality-problems.md) | Test discovery and reports, metadata and attachments, Problems API, PMD, plugin validation |
| [configuration-cache-modeling.md](references/configuration-cache-modeling.md) | Configuration Cache modes and diagnostics, lazy configurations and attributes, immutable collections |
| [dependencies-publishing-distributions.md](references/dependencies-publishing-distributions.md) | Dependency APIs, custom components, Maven POMs, archives, distributions, repository failures |
| [cli-tooling-platforms.md](references/cli-tooling-platforms.md) | Wrapper security and retries, console modes, diagnostics, Tooling API, TestKit, platform support |

## Upgrade to Gradle 9

Treat an upgrade as a runtime, language, and plugin-API migration rather than
only a wrapper version change.

1. Ensure the daemon can find JVM 17 or newer. Continue using toolchains for
   older compile, test, and worker targets where needed.
2. Audit Kotlin build logic for the Kotlin 2.2 language baseline and remove
   script-instance labels.
3. Recompile and test Groovy plugins against Groovy 4 behavior.
4. Check minimum compatible versions of the Kotlin, Android, and Develocity
   plugins before changing the wrapper.
5. Verify every included project maps to an existing writable directory.
6. Configure custom `Test` tasks explicitly and decide whether an empty test
   discovery result should fail.
7. Migrate removed conventions, process helpers, Kotlin DSL shortcuts,
   source-set types, permissions APIs, and toolchain constants.
8. Recheck archive reproducibility, artifact lifecycle wiring, publication
   mutation, and signing output.
9. Run with the Configuration Cache and inspect every reported problem.

See [migration-gradle-9.md](references/migration-gradle-9.md) for exact
replacements and behavioral details.

## Daemon JVM and toolchains

Do not confuse the JVM that runs Gradle with a Java toolchain used by tasks.
The daemon has its own runtime floor, selection criteria, provisioning, and
network behavior.

When a matching daemon JVM may not be installed:

```kotlin
plugins {
    id("org.gradle.toolchains.foojay-resolver-convention") version "0.9.0"
}
```

```text
./gradlew updateDaemonJvm --jvm-version=17 --jvm-vendor=adoptium
```

The generated `gradle/gradle-daemon-jvm.properties` can contain platform
download URLs in addition to the requested vendor and version.

For GraalVM Native Image workloads, request the capability instead of assuming
a particular installation:

```kotlin
java {
    toolchain {
        languageVersion = JavaLanguageVersion.of(21)
        nativeImageCapable = true
    }
}
```

Read [jvm-languages-build-logic.md](references/jvm-languages-build-logic.md)
before changing daemon startup, language plugins, precompiled scripts, or
plugin registrations.

## Configuration Cache

Configuration Cache behavior depends on when values are read and whether
listeners come from registered build services. Preserve those boundaries.

- Use read-only mode for jobs that may consume but must not populate entries.
- Use integrity checking only while diagnosing serialization; it increases
  entry size and makes cache reads and writes slower.
- Treat an execution-time cache problem as a build failure.
- Expect known unsupported features to fall back without using the cache, with
  the reason in the report.
- Do not rely on warning mode to preserve an entry for an incompatible task.
- Obtain `onTaskCompletion` listeners from registered build-service providers.
- Read project-property providers during execution when configuration should
  not depend on them.

For CI cache commands, keystore behavior, listener constraints, and precise
property tracking, see
[configuration-cache-modeling.md](references/configuration-cache-modeling.md).

## Test tasks and reports

Custom `Test` tasks need explicit inputs:

```kotlin
val test by testing.suites.existing(JvmTestSuite::class)
tasks.register<Test>("otherTest") {
    testClassesDirs = files(test.map { it.sources.output.classesDirs })
    classpath = files(test.map { it.sources.runtimeClasspath })
}
```

If test sources exist and no filters apply, discovering no tests fails by
default. Set `failOnNoDiscoveredTests = false` only when an empty result is an
intentional outcome.

For custom JUnit Platform engines, point `testDefinitionDirs` at non-class test
definitions. Use report metadata APIs for structured key-value data and
attachments, and use `TestMetadataListener` when build logic must consume those
events.

Report consumers should also tolerate millisecond timestamp precision, nested
class filenames containing `$`, source-specific aggregate report tabs, and
sortable HTML tables.

See [testing-quality-problems.md](references/testing-quality-problems.md) for
report structure, external test reporting, Problems API data, PMD formats, and
validation behavior.

## Lazy configuration and model APIs

Keep registrations and relationships lazy:

```kotlin
configurations {
    val parent = dependencyScope("parent")
    resolvable("child") {
        extendsFrom(parent)
    }
}
```

- Prefer `register` and role-based factories over `create`.
- Pass providers to configuration inheritance and publication variant APIs.
- Use `AttributeContainer.addAllLater` when the source must remain live.
- Remember that imported attributes override earlier destination values, but
  later destination values win.
- Use `DomainObjectCollection.disallowChanges()` to freeze membership without
  realizing lazy entries; contained objects remain mutable.

Read [configuration-cache-modeling.md](references/configuration-cache-modeling.md)
for the complete precedence and realization rules.

## Publishing and distributions

Creating a visible configuration does not automatically attach its artifacts
to `assemble` or `archives`; wire both lifecycle relationships explicitly.
When producing custom publications, obtain an ad hoc component from the
publishing extension and pass configuration providers to preserve laziness.

Archive tasks are reproducible by default. Re-enable filesystem order,
timestamps, and permissions only when a downstream consumer requires them.
Published distribution ZIPs have both checksum and signature sidecars, which
serve different verification purposes.

See
[dependencies-publishing-distributions.md](references/dependencies-publishing-distributions.md)
for component factories, Maven distribution management, EAR descriptors,
plugin publication requirements, and repository failure behavior.

## CLI, Wrapper, and Tooling API

For unattended builds, combine non-interactive operation with an intentional
console policy:

```text
./gradlew --non-interactive --console=plain build
```

`NO_COLOR` removes color while preserving other rich-console behavior.
`--console=colored` adds color to plain output without progress bars.

Wrapper download retries are opt-in. Bearer credentials take precedence over
Basic credentials, and credential use should be restricted by host. Verify
download signatures independently from checksums when establishing
authenticity.

Tooling API clients running on Java 25 should account for native-access
requirements. Clients can also configure independent action parallelism,
stream values, retrieve structured problem data, and request version or help
models without starting a daemon.

See [cli-tooling-platforms.md](references/cli-tooling-platforms.md) for exact
properties, reporting commands, Wrapper selectors, and platform caveats.

## Diagnostic starting points

| Symptom | First check |
| --- | --- |
| Custom test task runs zero tests | Explicit `testClassesDirs` and `classpath`, then framework discovery |
| Cache entry is unexpectedly discarded | Incompatible tasks, execution-time problems, and listener providers |
| Registered configuration realizes early | `create`, direct `get()`, or a non-provider relationship |
| Transform selection is ambiguous | Run `artifactTransforms` and compare input/output attributes |
| Published artifact is missing | Explicit `assemble`, `archives`, component, and variant wiring |
| Included project fails during configuration | Directory existence, type, and writability |
| Wrapper authentication reaches the wrong host | Host restrictions and bearer-versus-Basic precedence |
| Test report combines unrelated suites | Confirm each aggregate source has its own report tab |
| Tooling client behaves serially | `org.gradle.tooling.parallel` and its fallback value |
| Groovy build reads a parent property | Remove implicit lookup and enable the parent-lookup preview |

## Working principles

- Preserve Provider API laziness until a value is genuinely needed.
- Distinguish stable APIs from incubating or partially stable surfaces.
- Configure task inputs, outputs, lifecycle dependencies, and publication
  variants explicitly.
- Use structured Problems and test metadata instead of parsing console text.
- Treat authentication scope, signature verification, and repository failure
  handling as part of build correctness.
- Validate changes with the project wrapper, representative JDKs, the
  Configuration Cache, and the publication or report consumer affected.
