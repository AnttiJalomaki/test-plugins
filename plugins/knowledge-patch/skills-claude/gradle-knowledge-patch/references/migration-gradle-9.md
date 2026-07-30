# Gradle 9 Migration

Use this reference while upgrading a build or plugin. It consolidates the
breaking-change extraction identified by the full batch id
`9.0.0-upgrade`; related behavior introduced in `9.0.0` and the parent-project
lookup preview from `9.6.1` are included where they affect migration choices.

## Runtime, language, and plugin compatibility

### Separate daemon runtime from task toolchains

The Gradle 9 daemon requires JVM 17 or newer. Compilation, tests, and workers
can still target older JVMs through toolchains. The wrapper and command-line
launcher, Tooling API, and TestKit can themselves run on JVM 8, but the
launcher still has to locate a JVM 17+ daemon to execute the build.

### Update Kotlin build logic

Gradle embeds Kotlin 2.2.0 and uses Kotlin language version 2.2 for scripts,
build logic, and plugins. Kotlin language versions 1.4 through 1.7 are no
longer supported.

Script-instance labels such as `this@Build_gradle` no longer compile. Refer to
`project`, `settings`, or `gradle`, depending on the intended receiver.

Kotlin DSL plugins built with Gradle 9 require Gradle 8.11 or newer unless
they are explicitly compiled against Kotlin 1.x. Check the target Gradle
range when publishing a plugin from a Gradle 9 build.

### Rebuild Groovy plugins for Groovy 4

Gradle embeds Groovy 4.0.27, including package and module changes and
delegate-first dynamic lookup. Recompile and test Groovy plugins:

- Groovy 3-compiled code can fail when resolving `super` calls.
- A dynamically compiled inherited closure can lose access to private members
  of its parent.
- `@CompileStatic` avoids the inherited-closure lookup problem.

Groovy DSL plugins built with Gradle 9 require Gradle 7.0 or newer.

### Check ecosystem floors

Gradle 9 supports Kotlin Gradle Plugin 2.0.0 or newer, Android Gradle Plugin
8.4.0 or newer, and Gradle Enterprise Plugin 3.13.1 or newer. Resolve those
constraints before diagnosing downstream script errors.

## Project layout and native builds

Every project included from settings must map to an existing writable
directory. Configuration fails when an included path is missing, read-only,
or not a directory. Create and permission custom project directories before
the project is included.

The C++ and Swift plugins no longer use software-model plugin infrastructure.
Move `toolChains` out of `model {}` and configure it at the top level.

The custom settings and build-file layout switches are removed:

- `-c` and `--settings-file`
- `-b` and `--build-file`
- `GradleBuild.buildFile`

Adopt the standard settings/build-file layout rather than trying to redirect
these files.

## Task and test behavior

### Supply custom `Test` inputs

A newly registered `Test` task no longer inherits the built-in `test` source
set's test classes and runtime classpath. Without explicit inputs, an old
custom task can silently execute no tests.

```kotlin
val test by testing.suites.existing(JvmTestSuite::class)
tasks.register<Test>("otherTest") {
    testClassesDirs = files(test.map { it.sources.output.classesDirs })
    classpath = files(test.map { it.sources.runtimeClasspath })
}
```

Alternatively, model the additional target through JVM test suites.

When test sources exist and no filters apply, a task that discovers no tests
now fails. This exposes a mismatched test engine or framework. Set
`failOnNoDiscoveredTests = false` on `AbstractTestTask` only when no discovery
is valid for that task.

`ValidatePlugins` now depends on Java Toolchains infrastructure. Apply
`jvm-toolchains` explicitly if no Java or other JVM plugin supplies it:

```kotlin
plugins {
    id("jvm-toolchains")
}
```

### Account for updated tool defaults

Defaults are Checkstyle 10.24.0, CodeNarc 3.6.0, PMD 7.13.0, JUnit Jupiter
5.12.2, TestNG 7.11.0, and Spock 2.3. Gradle's JGit is 7.2.1 and can use an
SSH agent. Pin versions when a build cannot accept a changed default.

## Kotlin and Java API corrections

Gradle's public API uses JSpecify instead of JSR-305. Kotlin 2.1 and newer
enforce the generic bounds more precisely:

- Extensions on `Provider<T>` commonly need `T : Any`.
- `Property<String?>` is invalid.
- A nullable value cannot be passed where an API declares `Map<String, *>`.

```kotlin
fun <T : Any> Provider<T>.someExtension() = get()
```

Classes extending Gradle-provided classes with `@Inject` getters must be
abstract. `ConfigurationVariant.getDescription()` now returns
`Property<String>` rather than `Optional<String>`.

Do not make an exhaustive subtype assumption for `ComponentIdentifier`.
Handle `RootComponentIdentifier` and preserve a fallback for future unknown
subtypes.

## Removed conventions and direct replacements

The `Convention` API and `Project.getConvention()`/`Task.getConvention()` are
removed. Use extensions, configure `war` and `ear` tasks directly, and use the
`base` extension for base-plugin properties.

Apply these API replacements:

| Removed API | Replacement |
| --- | --- |
| `JvmVendorSpec.IBM_SEMERU` | `JvmVendorSpec.IBM` |
| `IdeaModule.testSourceDirs` | `testSources` |
| `IdeaModule.testResourceDirs` | `testResources` |
| `WriteProperties.outputFile` | `destinationFile` |
| `GroovySourceSet` | `GroovySourceDirectorySet` |
| `ScalaSourceSet` | `ScalaSourceDirectorySet` |
| Integer Unix-mode copy APIs | `FilePermissions` / `ConfigurableFilePermissions` |

`Project.exec`, `Project.javaexec`, and their script-level Kotlin and Groovy
counterparts are removed. Rewrite affected build logic and plugins so they no
longer use those helpers.

## Removed Kotlin DSL behavior

Replace removed shortcuts and eager access:

- Replace `"name"()` domain-object references with `named("name")`.
- Explicitly dereference providers instead of using eager provider accessors
  such as `configurations.compileClasspath.files`.
- Do not access `libraries` or `bundles` catalog entries inside `plugins {}`.
- Replace `kotlinDslPluginOptions.jvmTarget` with a Java toolchain.

The `` `gradle-enterprise` `` shorthand is removed. Apply Develocity by ID and
version:

```kotlin
plugins {
    id("com.gradle.develocity") version "4.0.2"
}
```

The deprecated `com.gradle.enterprise` ID is only a temporary bridge.

## Removed Groovy conveniences

`groovy-test`, `groovy-console`, and `groovy-sql` are no longer bundled or
provided by `localGroovy`; declare required modules explicitly.

`org.gradle.util.CollectionUtils`, `ConfigureUtil`, and
`ClosureBackedAction` are removed. Plugin DSLs should expose methods accepting
`Action` rather than Closure-specific API types.

## Archives, artifacts, publication, and signing

`Jar`, `Ear`, `War`, `Zip`, and other `AbstractArchiveTask` tasks now sort
entries deterministically, use fixed timestamps, and assign mode `0755` to
directories and `0644` to files.

Only restore filesystem metadata for a consumer that requires it:

```kotlin
tasks.withType<AbstractArchiveTask>().configureEach {
    isReproducibleFileOrder = false
    isPreserveFileTimestamps = true
    useFileSystemPermissions()
}
```

When `ear`, `war`, and `java` are combined, `assemble` builds all corresponding
artifacts and `archives` contains all of them. A custom visible configuration,
however, does not automatically join either lifecycle. Wire it explicitly:

```kotlin
tasks.named("assemble") {
    dependsOn(special.artifacts)
}
configurations.named("archives") {
    outgoing.artifact(specialJar)
}
```

Changing Gradle Module Metadata after an eagerly created publication has been
populated from the same component is now an error. Finish component metadata
configuration before populating publications.

The signing plugin emits the OpenPGP signature version matching the key,
including version 6; consumers must not assume every signature is version 4.

## Configuration Cache migration

With Configuration Cache enabled, `onTaskCompletion` listeners must be
providers created from a registered build service. Unsupported providers now
produce a cache problem instead of being ignored.

An incompatible task always discards the entry, even under
`org.gradle.configuration-cache.problems=warn`. The temporary escape hatch for
unsupported build-event listeners is:

```properties
org.gradle.configuration-cache.unsafe.ignore.unsupported-build-events-listeners=true
```

Use that property only while migrating the listener.

The following properties no longer control cleanup:

- `org.gradle.cache.cleanup` cannot disable cleanup.
- `buildCache.local.removeUnusedEntriesAfterDays` cannot set local build-cache
  retention.

Configure Gradle User Home cache cleanup through an init script.

## Parent-project lookup removal

Groovy DSL lookup of a missing property or method from a parent project is
deprecated. The same applies to `findProperty()`, `property()`, and
`hasProperty()` when their result comes from a parent. This lookup is scheduled
for removal in Gradle 10.

Replace implicit references with an explicit project or shared provider. After
the migration, make regressions fail early:

```kotlin
// settings.gradle.kts
enableFeaturePreview("NO_IMPLICIT_LOOKUP_IN_PARENT_PROJECTS")
```

## Version-selection expectation

Gradle 9 adopts `MAJOR.MINOR.PATCH` versioning. Older release names and
backports are unchanged. Internal and `@Incubating` features remain outside
the public semantic-versioning compatibility guarantee and may change in a
minor release.
