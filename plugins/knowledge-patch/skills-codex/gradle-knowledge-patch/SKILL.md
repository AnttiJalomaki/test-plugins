---
name: gradle-knowledge-patch
description: Gradle
version: 9.6.1
license: MIT
metadata:
  author: Nevaberry
---


# Gradle Knowledge Patch

Use this patch when writing, reviewing, upgrading, or troubleshooting Gradle
builds, plugins, TestKit integrations, or Tooling API clients. Start with the
quick references below, then open only the topic files relevant to the task.

## Reference index

| Reference | Topics |
| --- | --- |
| [configuration-cache-and-laziness.md](references/configuration-cache-and-laziness.md) | Configuration Cache behavior, integrity checks, providers, lazy configurations, attributes, and immutable collections |
| [daemon-cli-and-platforms.md](references/daemon-cli-and-platforms.md) | Daemon JVMs, Java and native platforms, Wrapper behavior, console modes, reports, distributions, and CLI diagnostics |
| [dependencies-publishing-and-distribution.md](references/dependencies-publishing-and-distribution.md) | Dependency APIs, artifact transforms, publishing, signing, distributions, repositories, ANTLR, and EAR |
| [gradle-9-upgrade.md](references/gradle-9-upgrade.md) | Gradle 9 runtime floors, language changes, removed APIs, task behavior, archives, and migration replacements |
| [testing-and-quality.md](references/testing-and-quality.md) | Test task inputs, discovery, reports, metadata, TestKit, PMD, validation, and default quality-tool versions |
| [tooling-problems-and-build-logic.md](references/tooling-problems-and-build-logic.md) | Tooling API, Problems API, plugin build logic, Kotlin and Groovy DSL behavior, layouts, and plugin IDs |

## Gradle 9 migration: fix these first

### Run the daemon on Java 17 or newer

Gradle 9 requires JVM 17+ for the daemon. Compilation, tests, and workers can
still target older Java versions through toolchains. A launcher or Tooling API
client may itself run on JVM 8, but it must be able to locate a JVM 17+ daemon.

Also verify the surrounding plugin floors:

- Kotlin Gradle Plugin 2.0.0 or newer.
- Android Gradle Plugin 8.4.0 or newer.
- Gradle Enterprise Plugin 3.13.1 or newer.
- Kotlin DSL plugins built with Gradle 9 normally require Gradle 8.11+.
- Groovy DSL plugins built with Gradle 9 require Gradle 7.0+.

### Update embedded-language assumptions

Gradle 9 embeds Kotlin 2.2.0 and uses Kotlin language version 2.2 in scripts,
build logic, and plugins. Remove script-instance labels such as
`this@Build_gradle`; use `project`, `settings`, or `gradle`.

Gradle 9 also embeds Groovy 4.0.27. Recompile Groovy plugins and test `super`
resolution and closure access to private parent members. `@CompileStatic`
avoids the changed delegate-first dynamic lookup in the latter case.

### Replace removed entry points

| Removed behavior | Replacement |
| --- | --- |
| `-c` / `--settings-file`, `-b` / `--build-file`, `GradleBuild.buildFile` | Use conventional settings and build locations |
| `Convention`, `getConvention()` | Extensions; configure `war`/`ear` tasks directly; use the `base` extension |
| `Project.exec`, `Project.javaexec`, script-level counterparts | Remove calls; these helpers no longer exist |
| `JvmVendorSpec.IBM_SEMERU` | `JvmVendorSpec.IBM` |
| `WriteProperties.outputFile` | `destinationFile` |
| `IdeaModule.testSourceDirs`, `testResourceDirs` | `testSources`, `testResources` |
| `GroovySourceSet`, `ScalaSourceSet` | `GroovySourceDirectorySet`, `ScalaSourceDirectorySet` |
| Integer Unix-mode copy APIs | `FilePermissions` / `ConfigurableFilePermissions` |
| `` `gradle-enterprise` `` Kotlin DSL shorthand | Explicit `com.gradle.develocity` plugin ID and version |
| `kotlinDslPluginOptions.jvmTarget` | A Java toolchain |
| `"name"()` domain-object shorthand | `named("name")` |
| Eager provider accessors | Explicitly dereference or compose providers |
| Closure-only plugin DSL entry points | `Action`-accepting methods |

`groovy-test`, `groovy-console`, and `groovy-sql` are no longer supplied by
`localGroovy`. The Gradle utility types `CollectionUtils`, `ConfigureUtil`, and
`ClosureBackedAction` are also removed.

### Recheck task and project assumptions

Every project included from settings must map to an existing, writable
directory. A custom `Test` task no longer inherits the built-in `test` source
set's inputs, so configure both `testClassesDirs` and `classpath` explicitly or
use a JVM test suite. Test tasks now fail when sources exist but no tests are
discovered; set `failOnNoDiscoveredTests = false` only for an intentional empty
result.

Move C++ and Swift `toolChains` configuration out of `model {}` and configure
it at the top level. Apply `jvm-toolchains` before using `ValidatePlugins` when
no JVM plugin already provides toolchain infrastructure.

### Recheck API types

Gradle API nullability uses JSpecify. Kotlin extensions over `Provider<T>`
commonly need `T : Any`; `Property<String?>` is invalid, and nullable values
cannot be supplied to APIs declaring `Map<String, *>`.

Injected getter subclasses of Gradle-provided classes must be abstract.
`ConfigurationVariant.getDescription()` now returns `Property<String>`.
Exhaustive logic over `ComponentIdentifier` must tolerate
`RootComponentIdentifier` and future unknown implementations.

## Configuration Cache essentials

Gradle 9 prefers Configuration Cache but does not force it. A compatible build
that has not enabled it receives a suggestion; set
`org.gradle.configuration-cache=false` to suppress that suggestion. Known
unsupported features cause a non-cache fallback with the reason in the report,
but a cache problem during task execution aborts immediately.

With Configuration Cache enabled:

- `onTaskCompletion` listeners must be providers from registered build
  services.
- Unsupported providers are cache problems.
- Incompatible tasks discard the cache entry even with
  `org.gradle.configuration-cache.problems=warn`.
- The temporary listener escape hatch is
  `org.gradle.configuration-cache.unsafe.ignore.unsupported-build-events-listeners=true`.

For cache consumers that must never populate entries:

```text
./gradlew --configuration-cache \
  -Dorg.gradle.configuration-cache.read-only=true build
```

Use `org.gradle.configuration-cache.integrity-check=true` only while
troubleshooting serialization; it increases entry size and slows cache I/O.

Project properties supplied with `-Dorg.gradle.project.<name>` or
`ORG_GRADLE_PROJECT_<name>` invalidate an entry only when read during
configuration. A provider read only during task execution can observe a new
value while the existing entry is reused.

## Preserve lazy configuration

Prefer `register` and role-based configuration factories over eager `create`.
Applying `base`, directly or through a JVM plugin, no longer realizes all
registered configurations.

Use the newer lazy composition APIs where appropriate:

```kotlin
val parent = configurations.dependencyScope("parent")
configurations.resolvable("child") {
    extendsFrom(parent)
}

target.attributes.addAllLater(source.attributes)
```

`addAllLater` tracks later source changes. Imported attributes override values
already in the destination; destination values set afterward win.

Publishing variant methods accept
`Provider<ConsumableConfiguration>`, delaying realization until publication.
`DomainObjectCollection.disallowChanges()` freezes membership without realizing
lazy entries; contained objects remain mutable.

## Testing essentials

When a custom JUnit Platform engine discovers non-class definitions, point
`Test.testDefinitionDirs` at them. This supports inputs such as Cucumber
feature files without a placeholder suite class.

JUnit Platform `TestReporter` key-value data and files appear in HTML and XML
reports. Build logic can consume the same structured events through
`Test.addTestMetadataListener(TestMetadataListener)`.

Custom test infrastructure can inject `TestEventReporterFactory` and report
nested groups, tests, timestamped metadata, Gradle binary results, and HTML
reports. For high-volume TestKit output, stream and close
`BuildResult.getOutputReader()` instead of materializing `getOutput()`.

Test reports now retain framework hierarchy and attribute output to the test
that emitted it. Aggregate reports keep each input source on its own tab rather
than merging overlapping structures. HTML result columns are sortable.

## Toolchains, Wrapper, and unattended execution

Daemon JVM criteria can auto-provision a matching JDK when a resolver such as
Foojay is installed. Generate `gradle/gradle-daemon-jvm.properties` with
`updateDaemonJvm`; it records vendor/version criteria and per-platform URLs.
Daemon toolchains are stable APIs.

Toolchains can require GraalVM Native Image:

```kotlin
java {
    toolchain {
        languageVersion = JavaLanguageVersion.of(21)
        nativeImageCapable = true
    }
}
```

Gradle 9 Wrapper version selectors may be partial:

```text
./gradlew wrapper --gradle-version=9
./gradlew wrapper --gradle-version=9.1
```

Do not apply that interpretation to pre-9 versions such as `8.12`, which is an
exact historical release.

For unattended runs use `--non-interactive` or set
`org.gradle.console.interactive=false`. A non-empty `NO_COLOR` suppresses color
while preserving other rich-console behavior. `--console=colored` provides
color without progress bars.

## Publishing and artifact lifecycles

When `ear`, `war`, and `java` are combined, `assemble` builds all three artifact
types and `archives` contains them. A custom visible configuration does not
join either lifecycle automatically; wire its task into `assemble` and its
artifact into `archives`.

Do not mutate Gradle Module Metadata after an eagerly created publication has
been populated from the same component; this is an error. OpenPGP signatures
now match the key version, including version 6 keys.

The publishing extension exposes `SoftwareComponentFactory`, and ad hoc
components can consume lazy configuration providers. Maven publications can
also declare POM distribution management directly.

## Diagnostics worth reaching for

- `./gradlew artifactTransforms` lists registered artifact transforms,
  cacheability, and input/output attributes.
- `./gradlew ... --task-graph` prints requested-task dependencies without
  executing them.
- `./gradlew tasks --provenance` shows where tasks were registered.
- `./gradlew test --warning-mode=all` renders relevant Problems API entries in
  the console while retaining the HTML report link.
- `BuildEnvironment.getVersionInfo()` returns exact version output without
  starting a daemon; the Tooling API `Help` model returns rendered help.
- `ProjectLayout.settingsDirectory` resolves files relative to the settings
  script without reaching through `rootProject`.

For complete constraints, edge cases, and examples, open the relevant topic
file from the reference index before changing a build.
