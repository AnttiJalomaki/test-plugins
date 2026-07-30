# Dependencies, Publishing, and Distributions

## Strongly typed dependency blocks

The `Dependencies` API used for plugin-defined, strongly typed `dependencies`
blocks is partially stable as of `8.13.0`. Version-catalog dependencies were
not included in that stability promotion and remain subject to review.

## Detached and project dependencies

A detached configuration can resolve a dependency on its own project as of
`9.0.0`. Temporary resolution-only configurations may therefore include
project dependencies rather than only externally identified components.

`Project.getDependencyFactory()` became a stable API in `9.1.0` and is covered
by Gradle's backward-compatibility guarantees.

## Lazy configurations and published variants

`AdhocComponentWithVariants.addVariantsFromConfiguration(...)` and
`withVariantsFromConfiguration(...)` accept
`Provider<ConsumableConfiguration>` as of `9.2.0`. Pass the provider so the
configuration is realized only when its publication is published:

```kotlin
val publishedVariant =
    configurations.consumable("publishedVariant")

publishing {
    val component =
        softwareComponentFactory.adhoc("custom")
    component.addVariantsFromConfiguration(publishedVariant) {}
}
```

## Custom publishable components

The `publishing` extension exposes `SoftwareComponentFactory` directly (since
`9.2.0`). A build or plugin can create an ad hoc component without applying a
JVM plugin merely to obtain one:

```kotlin
publishing {
    val component =
        softwareComponentFactory.adhoc("custom")
    component.addVariantsFromConfiguration(
        consumableConfiguration
    ) {}
    publications {
        create<MavenPublication>("maven") {
            from(component)
        }
    }
}
```

## Maven POM distribution management

`MavenPublication` can declare a distribution repository and emit it in the
generated POM (since `9.1.0`):

```kotlin
publications.withType<MavenPublication>().configureEach {
    pom {
        distributionManagement {
            repository {
                id = "github"
                url =
                    "https://maven.pkg.github.com/OWNER/REPOSITORY"
            }
        }
    }
}
```

## Plugin publication

Plugin Publishing plugin 2.0.0 supports the Configuration Cache and exposes
configuration through the Provider API (`9.1.0`). It requires Gradle 7.4 or
newer. Signed publications need Gradle 8.1.1 or newer for full Configuration
Cache compatibility.

## Distribution plugins

The `distribution-base` plugin provides distribution capabilities without
creating a default `main` distribution (since `8.13.0`). The `distribution`
plugin wraps it and additionally creates `main`.

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

Use the base form when all distributions should be explicitly named.

## EAR descriptors

The EAR plugin can generate Jakarta EE 11 deployment descriptors directly
(since `9.1.0`):

```kotlin
tasks.ear {
    deploymentDescriptor {
        version = "11"
    }
}
```

A custom descriptor file is not required solely to select Jakarta EE 11.

## Distribution verification

Every published Gradle distribution ZIP has an ASCII-armored `.asc` signature
beside its `.sha256` checksum as of `9.3.0`, allowing authenticity verification
in addition to checksum-based integrity checking.

## Repository failure behavior

After repeated retrieval failures or a fatal error such as an incorrect
hostname, Gradle disables that repository for the remainder of the build
(since `9.3.0`). Resolution normally fails at that point rather than continuing
to later repositories unless continuation was configured.

Do not assume repository ordering alone provides failover. Correct fatal
repository configuration and make any intended continuation policy explicit.
