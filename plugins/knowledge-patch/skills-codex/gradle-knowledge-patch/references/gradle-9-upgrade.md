# Gradle 9 Upgrade

Use this reference before changing the Wrapper to Gradle 9 or rebuilding a
plugin with Gradle 9. The migration items here come from the
`9.0.0-upgrade` batch.

## Update runtime and plugin compatibility

### Run the daemon on Java 17+

The Gradle 9 daemon requires JVM 17 or newer. Compilation, tests, and workers
may still target older JVMs through toolchains. The Wrapper and command-line
launcher, Tooling API, and TestKit remain able to run on JVM 8, but a launcher
must locate a JVM 17+ daemon to execute the build.

### Check plugin compatibility floors

- Kotlin Gradle Plugin: 2.0.0+
- Android Gradle Plugin: 8.4.0+
- Gradle Enterprise Plugin: 3.13.1+

Kotlin DSL plugins built with Gradle 9 require Gradle 8.11+ unless explicitly
compiled against Kotlin 1.x. Groovy DSL plugins built with Gradle 9 require
Gradle 7.0+.

## Migrate Kotlin DSL and API nullability

Gradle 9 embeds Kotlin 2.2.0 and uses language version 2.2 for scripts, build
logic, and plugins. Kotlin language versions 1.4 through 1.7 are no longer
supported.

Script-instance labels such as `this@Build_gradle` no longer compile. Use
`project`, `settings`, or `gradle` as appropriate.

The public Gradle API now uses JSpecify rather than JSR-305. Kotlin 2.1+
enforces generic bounds more precisely:

- Extensions on `Provider<T>` commonly need `T : Any`.
- `Property<String?>` is invalid.
- A nullable value cannot be passed where an API declares `Map<String, *>`.

```kotlin
fun <T : Any> Provider<T>.someExtension() = get()
```

Remove these Kotlin DSL shortcuts:

- `"name"()` domain-object references; use `named("name")`.
- Eager provider accessors such as
  `configurations.compileClasspath.files`; explicitly dereference providers.
- `libraries` and `bundles` version-catalog access inside `plugins {}`; those
  entries are not available there.
- `kotlinDslPluginOptions.jvmTarget`; use a Java toolchain.

The Kotlin DSL `` `gradle-enterprise` `` plugin shorthand is removed. Apply the
renamed plugin with an explicit ID and version, or temporarily use the
deprecated `com.gradle.enterprise` ID:

```kotlin
plugins {
    id("com.gradle.develocity") version "4.0.2"
}
```

## Recompile and adjust Groovy code

Gradle 9 embeds Groovy 4.0.27, including its package/module changes and
delegate-first dynamic lookup order.

- Recompile Groovy plugins. Code compiled with Groovy 3 can fail to resolve
  `super` calls.
- A dynamically compiled closure inherited from a parent can lose access to
  the parent's private members. `@CompileStatic` avoids this lookup issue.
- `groovy-test`, `groovy-console`, and `groovy-sql` are no longer bundled or
  provided by `localGroovy`.
- `org.gradle.util.CollectionUtils`, `ConfigureUtil`, and
  `ClosureBackedAction` are removed. Plugin DSLs should expose methods that
  accept `Action` rather than Closure-specific APIs.

## Move native toolchains out of the software model

The C++ and Swift plugins no longer use software-model plugin infrastructure.
Move `toolChains` configuration out of `model {}` and configure it at the top
level of the build script.

## Validate included projects and plugins

Every project included from settings must map to a directory that exists, is
writable, and is actually a directory. Gradle fails during configuration when
an included path violates any of these rules.

`ValidatePlugins` now requires Java Toolchains infrastructure. Apply
`jvm-toolchains` explicitly if no Java or other JVM plugin supplies it:

```kotlin
plugins {
    id("jvm-toolchains")
}
```

Classes extending Gradle-provided classes with `@Inject` getters must be
abstract.

Other plugin API changes:

- `ConfigurationVariant.getDescription()` returns `Property<String>` rather
  than `Optional<String>`.
- Code switching on `ComponentIdentifier` must tolerate
  `RootComponentIdentifier` and future unknown subtypes.

## Reconfigure custom tests

A newly registered `Test` task no longer inherits classes or the runtime
classpath from the built-in `test` source set. A task relying on the old
convention silently runs no tests. Configure both inputs explicitly:

```kotlin
val test by testing.suites.existing(JvmTestSuite::class)
tasks.register<Test>("otherTest") {
    testClassesDirs = files(test.map { it.sources.output.classesDirs })
    classpath = files(test.map { it.sources.runtimeClasspath })
}
```

Creating the extra target through JVM test suites is the other supported
approach.

When test sources exist and no filters apply, a task that discovers no tests
now fails. Set `failOnNoDiscoveredTests = false` on `AbstractTestTask` only
when an empty discovery result is intentional.

## Account for reproducible archive defaults

`Jar`, `Ear`, `War`, `Zip`, and other `AbstractArchiveTask` tasks now:

- sort entries deterministically;
- use fixed timestamps;
- assign `0755` to directories;
- assign `0644` to files.

Opt back into filesystem metadata only when a consumer requires it:

```kotlin
tasks.withType<AbstractArchiveTask>().configureEach {
    isReproducibleFileOrder = false
    isPreserveFileTimestamps = true
    useFileSystemPermissions()
}
```

## Wire artifacts into lifecycles

When `ear`, `war`, and `java` are combined, `assemble` now builds all
corresponding artifacts and `archives` contains all of them.

A custom visible configuration no longer makes its outgoing artifact part of
either lifecycle automatically. Wire both explicitly:

```kotlin
tasks.named("assemble") {
    dependsOn(special.artifacts)
}
configurations.named("archives") {
    outgoing.artifact(specialJar)
}
```

## Finalize publication metadata before population

Changing Gradle Module Metadata after an eagerly created publication has been
populated from the same component is now an error instead of a warning.

The signing plugin produces an OpenPGP signature version matching the key
version. A version 6 key therefore produces a version 6 signature rather than
always producing version 4.

## Make Configuration Cache listeners compatible

With Configuration Cache enabled:

- `onTaskCompletion` listeners must be providers created from a registered
  build service.
- Unsupported providers cause a cache problem instead of being ignored.
- Incompatible tasks discard the cache entry even with
  `org.gradle.configuration-cache.problems=warn`.

The temporary escape hatch for listeners is:

```properties
org.gradle.configuration-cache.unsafe.ignore.unsupported-build-events-listeners=true
```

## Update bundled tool expectations

The Gradle 9 defaults are:

| Tool | Default |
| --- | --- |
| Checkstyle | 10.24.0 |
| CodeNarc | 3.6.0 |
| PMD | 7.13.0 |
| JUnit Jupiter | 5.12.2 |
| TestNG | 7.11.0 |
| Spock | 2.3 |

Gradle's JGit is 7.2.1 and can use an SSH agent.

## Remove custom build layouts and conventions

The following are removed:

- `-c` / `--settings-file`
- `-b` / `--build-file`
- `GradleBuild.buildFile`
- `Convention`
- `Project.getConvention()`
- `Task.getConvention()`

Migrate convention data to extensions, configure `war` and `ear` tasks
directly, and use the `base` extension for base-plugin properties.

## Configure cache cleanup in Gradle User Home

`org.gradle.cache.cleanup` no longer disables cleanup.
`buildCache.local.removeUnusedEntriesAfterDays` no longer controls local
build-cache retention. Configure both through the Gradle User Home
cache-cleanup settings in an init script.

## Apply direct API replacements

| Removed API | Replacement |
| --- | --- |
| `JvmVendorSpec.IBM_SEMERU` | `JvmVendorSpec.IBM` |
| `IdeaModule.testSourceDirs` | `testSources` |
| `IdeaModule.testResourceDirs` | `testResources` |
| `WriteProperties.outputFile` | `destinationFile` |
| `GroovySourceSet` | `GroovySourceDirectorySet` |
| `ScalaSourceSet` | `ScalaSourceDirectorySet` |
| Integer Unix-mode copy APIs | `FilePermissions` / `ConfigurableFilePermissions` |

## Remove process helper calls

`Project.exec`, `Project.javaexec`, and their script-level Kotlin and Groovy
counterparts are removed. Plugins and build logic can no longer launch
processes through those helpers.
