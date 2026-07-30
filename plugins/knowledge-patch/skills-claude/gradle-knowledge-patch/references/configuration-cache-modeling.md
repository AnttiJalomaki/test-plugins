# Configuration Cache and Lazy Modeling

## Enablement, fallback, and failures

Gradle 9 prefers the Configuration Cache but does not require it (`9.0.0`).
A compatible build that has not enabled it receives an end-of-build
suggestion. Suppress the suggestion by making the decision explicit:

```properties
org.gradle.configuration-cache=false
```

Known unsupported features cause an automatic non-cache fallback, with the
reason recorded in the Configuration Cache report. A cache problem encountered
during task execution aborts immediately; the task is not left up-to-date or
cached.

## Read-only mode for cache consumers

Read-only mode reuses an existing entry on a hit but never writes an entry
(since `9.1.0`):

```text
./gradlew --configuration-cache \
  -Dorg.gradle.configuration-cache.read-only=true build
```

This suits CI jobs such as pull-request builds that may consume shared entries
but must not populate them.

## Encryption keystore selection

Configuration Cache encryption uses the JVM's default keystore type when that
type supports symmetric keys (since `9.1.0`). Gradle falls back to `PKCS12` for
known asymmetric-only types. This matters on customized or FIPS-oriented JVM
security configurations; do not assume the keystore is always PKCS12.

## Integrity diagnostics

Set the integrity-check property for stricter serialization checking and more
precise cache-load diagnostics (since `8.14.0`):

```properties
org.gradle.configuration-cache.integrity-check=true
```

It increases cache size and slows reads and writes, so enable it only during
troubleshooting.

## Precise project-property tracking

Project properties supplied through
`-Dorg.gradle.project.<name>` or `ORG_GRADLE_PROJECT_<name>` no longer
invalidate an entry if configuration did not read that property (since
`9.6.1`).

A provider read only during task execution can observe a changed value while
Gradle reuses the existing configuration entry:

```kotlin
tasks.register("printValue") {
    val value = providers.gradleProperty("value").orElse("N/A")
    doLast { println(value.get()) }
}
```

Do not call `get()` during configuration if the property should remain an
execution-time input.

## Lazy configuration registration and inheritance

Applying `base`, directly or through Java or Kotlin plugins, does not realize
every configuration declared using `register` or a role-based factory such as
`resolvable` (since `8.14.0`):

```kotlin
configurations {
    register("myLazyConfiguration")
}
```

Prefer registration to `create` when immediate realization is unnecessary.

`Configuration.extendsFrom` accepts a `Provider<Configuration>` as of `9.4.0`,
so a lazily registered parent does not need `get()`:

```kotlin
configurations {
    val parent = dependencyScope("parent")
    resolvable("child") {
        extendsFrom(parent)
    }
}
```

## Lazy attribute merging

`AttributeContainer.addAllLater(source)` lazily imports an entire source
container (since `9.1.0`):

```kotlin
target.attributes.addAllLater(source.attributes)
```

Precedence and timing are significant:

1. Imported values override values already present on the destination.
2. Later changes in the source remain visible.
3. Values set on the destination after `addAllLater` take precedence.

## Artifact-transform diagnostics

Run the `artifactTransforms` report task to list all transforms registered in a
project (since `8.13.0`):

```text
./gradlew artifactTransforms
```

The report includes action type, cacheability, and input and output attributes.
Use it to inspect plugin registrations and diagnose ambiguous transforms.

## Immutable collection membership

`DomainObjectCollection.disallowChanges()` prevents later additions and
removals without realizing lazily added entries (since `9.5.0`):

```kotlin
val items = objects.domainObjectContainer(MyType::class)
val main = MyType("main")
items.add(main)
items.disallowChanges()
main.setFoo("bar")
```

The collection's membership is locked; contained objects remain mutable.

## Develocity without project configuration

Publish a Build Scan to a specified Develocity server without adding project
configuration by supplying the command-line URL (since `9.5.0`):

```text
./gradlew --develocity-url https://develocity.example.com build
```
