# Dependencies, Publishing, and Distributions

Use this reference when declaring dependencies, inspecting transforms,
constructing publishable components, wiring artifact lifecycles, generating
POMs, or configuring distributions and generated sources.

## Declare dependencies with promoted APIs

### Strongly typed dependency blocks

The `Dependencies` API used by plugin-defined, strongly typed `dependencies`
blocks became partially stable in `8.13.0`. Version-catalog dependencies were
not included in that stability promotion and remain under review.

### Kotlin DSL declaration helpers

In `9.0.0`, Kotlin DSL dependency and constraint `invoke` overloads became
stable. This includes overloads on named configuration providers and overloads
accepting `Provider` or `ProviderConvertible` values.

These related APIs are stable as well:

- `DependencyHandler.create(String, action)`
- `PluginDependenciesSpec.embeddedKotlin(String)`
- `GroovyBuilderScope.hasProperty(String)`

### Project dependency factory

`Project.getDependencyFactory()` is a stable, backward-compatible API starting
in `9.1.0`. Plugin code can use it instead of relying on internal factories.

### Detached configurations

A detached configuration can resolve a dependency on its own project as of
`9.0.0`. This allows temporary, resolution-only configurations to contain
project dependencies instead of only externally identified components.

## Select Scala explicitly

The `scala` and `scala-base` plugins accept `scalaVersion` in the `scala`
extension (`8.13.0`) and resolve the toolchain dependencies automatically:

```kotlin
scala {
    scalaVersion = "3.6.3"
}
```

Do not add `scala-library` solely to make Gradle infer the Scala version.

## Diagnose artifact transforms

The `artifactTransforms` report (`8.13.0`) lists every transform registered in
a project, including its action type, cacheability, and input/output
attributes:

```text
./gradlew artifactTransforms
```

Use it to audit plugin registrations and diagnose ambiguous transforms.

## Create publishable software components

The `publishing` extension exposes `SoftwareComponentFactory` directly as of
`9.2.0`. A plugin or build can create an ad hoc component without applying a
JVM plugin merely to obtain the factory:

```kotlin
publishing {
    val component = softwareComponentFactory.adhoc("custom")
    component.addVariantsFromConfiguration(consumableConfiguration) {}
    publications {
        create<MavenPublication>("maven") {
            from(component)
        }
    }
}
```

`AdhocComponentWithVariants.addVariantsFromConfiguration(...)` and
`withVariantsFromConfiguration(...)` also accept a
`Provider<ConsumableConfiguration>` (`9.2.0`). Passing the provider preserves
lazy configuration and realizes the configuration only when its publication
is actually published:

```kotlin
val publishedVariant = configurations.consumable("publishedVariant")

publishing {
    val component = softwareComponentFactory.adhoc("custom")
    component.addVariantsFromConfiguration(publishedVariant) {}
}
```

## Generate Maven distribution management

The `maven-publish` plugin can declare a distribution repository through
`MavenPublication` and emit it into the generated POM (`9.1.0`):

```kotlin
publications.withType<MavenPublication>().configureEach {
    pom {
        distributionManagement {
            repository {
                id = "github"
                url = "https://maven.pkg.github.com/OWNER/REPOSITORY"
            }
        }
    }
}
```

## Keep publication mutation and signing valid

The `9.0.0-upgrade` changes make it an error to change Gradle Module Metadata
after an eagerly created publication has been populated from the same
component. Finish component metadata configuration before eagerly creating or
populating the publication.

The signing plugin emits an OpenPGP signature version that matches the key
version, including version 6. Do not assume all generated signatures use
version 4.

Plugin Publishing plugin 2.0.0 (`9.1.0`) supports Configuration Cache and
exposes configuration through the Provider API. It requires Gradle 7.4+;
signed publications need Gradle 8.1.1+ for full Configuration Cache
compatibility.

## Wire artifact lifecycles explicitly

After the `9.0.0-upgrade`, combining `ear`, `war`, and `java` makes `assemble`
build all corresponding artifacts and puts all of them in `archives`.

A custom visible configuration has the opposite behavior: its outgoing
artifact is not automatically part of `assemble` or `archives`. Wire both:

```kotlin
tasks.named("assemble") {
    dependsOn(special.artifacts)
}
configurations.named("archives") {
    outgoing.artifact(specialJar)
}
```

## Build distributions without a default `main`

The `distribution-base` plugin (`8.13.0`) provides distribution capabilities
without creating a `main` distribution. The existing `distribution` plugin
wraps it and adds `main`.

```kotlin
plugins {
    id("distribution-base")
}

distributions {
    create("custom") {
        distributionBaseName = "customName"
        contents {
            from("src/customLocation")
        }
    }
}
```

Gradle distribution ZIPs have an ASCII-armored `.asc` signature beside the
`.sha256` checksum beginning in `9.3.0`, enabling authenticity checks as well
as checksum integrity checks.

## Generate ANTLR sources correctly

`AntlrTask.packageName` (`9.1.0`) sets both ANTLR 4's `-package` argument and
the matching output directory:

```kotlin
tasks.named("generateGrammarSource").configure {
    packageName = "com.example.generated"
}
```

It is limited to ANTLR 4. Passing `-package` directly is deprecated and becomes
an error in Gradle 10.

Changing the ANTLR generated-sources directory now updates the associated Java
source set automatically and wires consumers to depend on the generation task
(`9.1.0`). In `9.2.0`, tasks such as `generateGrammarSource` and
`generateTestGrammarSource` moved into the `Antlr` task group, so they appear
in ordinary `./gradlew tasks` output rather than only `tasks --all`.

## Generate Jakarta EE EAR descriptors

The EAR plugin can generate Jakarta EE 11 deployment descriptors directly
(`9.1.0`):

```kotlin
tasks.ear {
    deploymentDescriptor {
        version = "11"
    }
}
```

A separate custom descriptor file is no longer required for this version.

## Handle repository failure policy

As of `9.3.0`, repeated retrieval failures or fatal errors such as an incorrect
hostname disable that repository for the rest of the build. Dependency
resolution normally fails at that point instead of continuing through later
repositories unless continuation has been configured.
