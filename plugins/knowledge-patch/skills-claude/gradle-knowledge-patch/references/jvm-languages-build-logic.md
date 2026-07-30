# JVM, Languages, and Build Logic

## Daemon JVM selection and provisioning

When no installed JDK matches daemon JVM criteria, Gradle can download one
(since `8.13.0`). Apply Foojay resolver plugin 0.9.0 or a custom resolver, then
run `updateDaemonJvm`:

```kotlin
plugins {
    id("org.gradle.toolchains.foojay-resolver-convention") version "0.9.0"
}
```

```text
./gradlew updateDaemonJvm --jvm-version=17 --jvm-vendor=adoptium
```

The generated `gradle/gradle-daemon-jvm.properties` records the requested
vendor and version and can include a download URL for each platform.

Daemon toolchains became stable in `9.2.0`; configuring daemon JVM criteria no
longer emits an incubation warning.

`JAVA_HOME` participates in Java toolchain auto-detection as of `9.0.0`, which
aligns command-line discovery with IDE discovery.

Daemon logs older than 14 days are automatically deleted when a daemon shuts
down (since `9.4.0`).

For machines where address auto-detection is unsuitable, set the bind address
explicitly (since `9.5.0`):

```text
GRADLE_DAEMON_BIND_ADDRESS=192.168.1.10 ./gradlew build
```

This selects the address for both client-daemon and cross-daemon
communication.

## Java runtime and toolchain capabilities

Java and daemon toolchain selection can require a JDK that supplies GraalVM
Native Image (since `8.14.0`):

```kotlin
java {
    toolchain {
        languageVersion = JavaLanguageVersion.of(21)
        nativeImageCapable = true
    }
}
```

Gradle can run its daemon on Java 25 and use Java 25 toolchains as of `9.1.0`.
Tooling API clients on Java 25 must enable native access at startup because
the API uses JNI. Third-party tool compatibility may lag.

Daemon and toolchain support extends to Java 26 as of `9.4.0`; verify
third-party plugins and tools separately.

## Scala configuration

The `scala` and `scala-base` plugins accept an explicit `scalaVersion` in the
Scala extension (since `8.13.0`) and resolve the required Scala toolchain
dependencies automatically:

```kotlin
scala {
    scalaVersion = "3.6.3"
}
```

A `scala-library` dependency is no longer needed solely to select or infer the
Scala version.

## Kotlin scripts and convention plugins

Stable Kotlin DSL dependency and constraint `invoke` overloads are available
as of `9.0.0`. This includes overloads on named configuration providers and
overloads accepting `Provider` or `ProviderConvertible` values.
`DependencyHandler.create(String, action)`,
`PluginDependenciesSpec.embeddedKotlin(String)`, and
`GroovyBuilderScope.hasProperty(String)` are stable as well.

Precompiled Kotlin script plugins can apply and configure plugins supplied as
`compileOnly` dependencies, including their type-safe extension accessors
(since `9.1.0`).

Precompiled `*.settings.gradle.kts` plugins receive generated type-safe
accessors when the convention-plugin build applies `kotlin-dsl` (since
`9.5.0`):

```kotlin
// build-logic/build.gradle.kts
plugins {
    `kotlin-dsl`
}
```

```kotlin
// build-logic/src/main/kotlin/conventions.settings.gradle.kts
plugins {
    id("com.gradle.develocity")
}
develocity {
    buildScan {
        publishing.onlyIf { false }
    }
}
```

Settings extensions can then use typed configuration instead of string-based
lookups.

## Groovy lazy-property coercion

The Groovy DSL can coerce a string assigned to `Property<File>`,
`RegularFileProperty`, or `DirectoryProperty`, resolving the path relative to
the project directory (since `9.6.1`). It can also assign a scalar or array
directly to `ListProperty<T>` or `SetProperty<T>`:

```groovy
task.workingDir = '../my-build'
task.filter.includePatterns = 'Foo'
task.filter.includePatterns = ['Foo', 'Bar'] as String[]
```

Use these assignments only where the property element type makes the intended
coercion unambiguous.

## ANTLR source generation

For ANTLR 4, set the target package through `AntlrTask.packageName` (since
`9.1.0`):

```kotlin
tasks.named("generateGrammarSource").configure {
    packageName = "com.example.generated"
}
```

This supplies ANTLR's `-package` argument and selects the corresponding output
directory. Passing `-package` directly is deprecated and becomes an error in
Gradle 10.

Changing the generated-sources directory now updates the associated Java
source set and automatically wires its consumers to depend on the generation
task (`9.1.0`).

ANTLR tasks including `generateGrammarSource` and
`generateTestGrammarSource` belong to the `Antlr` task group as of `9.2.0`,
so they appear in normal `./gradlew tasks` output.

## Plugin registrations and model APIs

With `java-gradle-plugin`, a plugin registration uses its registration name as
the plugin ID unless `id` is set explicitly (since `9.4.0`):

```kotlin
gradlePlugin {
    plugins {
        register("my.plugin-id") {
            implementationClass = "my.PluginClass"
        }
    }
}
```

`ProjectLayout.settingsDirectory` exposes the directory containing
`settings.gradle` or `settings.gradle.kts` (since `8.13.0`):

```kotlin
val versionFile = layout.settingsDirectory.file("version.txt")
```

Use it for build-wide files instead of reaching through `rootProject`.

Plugin authors can call `DomainObjectCollection.disallowChanges()` to block
later additions and removals without realizing lazy entries (since `9.5.0`).
The lock covers membership only; properties of objects already in the
collection can still change.
