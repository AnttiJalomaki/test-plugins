# Configuration Cache and Lazy Configuration

Use this reference when making a build cache-compatible, diagnosing cache
serialization, or avoiding premature realization of Gradle domain objects.

## Choose Configuration Cache behavior

### Preferred, but optional

In Gradle `9.0.0`, Configuration Cache is preferred but remains optional.
Compatible builds that have not enabled it receive an end-of-build suggestion.
Suppress the suggestion by making the choice explicit:

```properties
org.gradle.configuration-cache=false
```

Known unsupported features cause an automatic non-cache fallback. The report
records the reason. A cache problem encountered during task execution instead
aborts immediately; it does not leave the task up-to-date or cached.

### Consume without populating

Read-only mode (`9.1.0`) reuses an existing entry on a hit but never writes a
new entry. It is useful for pull-request or other CI jobs that may consume a
shared cache but must not populate it:

```text
./gradlew --configuration-cache \
  -Dorg.gradle.configuration-cache.read-only=true build
```

### Select the encryption keystore

Since `9.1.0`, Configuration Cache encryption uses the JVM's default keystore
type when it supports symmetric keys. Gradle falls back to `PKCS12` for known
asymmetric-only formats. This improves behavior on JVMs with customized or
FIPS-oriented security configuration.

## Diagnose cache serialization

The integrity check introduced in `8.14.0` performs stricter serialization
checks and produces more precise cache-load diagnostics:

```properties
org.gradle.configuration-cache.integrity-check=true
```

Enable it only while troubleshooting. It makes cache entries larger and slows
both reads and writes.

## Register supported completion listeners

The `9.0.0-upgrade` changes make task-completion listener handling strict:

- `onTaskCompletion` listeners must be providers created from a registered
  build service.
- An unsupported provider is a Configuration Cache problem rather than being
  silently ignored.
- An incompatible task always discards the entry, even when
  `org.gradle.configuration-cache.problems=warn`.
- The temporary escape hatch for unsupported build-event listeners is
  `org.gradle.configuration-cache.unsafe.ignore.unsupported-build-events-listeners=true`.

Treat the escape hatch as migration scaffolding, not the final design.

## Track environment-backed project properties precisely

As of `9.6.1`, project properties supplied as
`-Dorg.gradle.project.<name>` system properties or
`ORG_GRADLE_PROJECT_<name>` environment variables invalidate an entry only
when the property was read during configuration.

Capture a provider during configuration and consume it during task execution
to let a reused entry observe the new value:

```kotlin
tasks.register("printValue") {
    val value = providers.gradleProperty("value").orElse("N/A")
    doLast { println(value.get()) }
}
```

Do not call `get()` during configuration if the value is intended to remain an
execution-time input.

## Register configurations lazily

Applying the `base` plugin, directly or indirectly through Java or Kotlin
plugins, no longer realizes every configuration declared with `register` or a
role-based factory (`8.14.0`).

```kotlin
configurations {
    register("myLazyConfiguration")
}
```

Prefer `register`, `dependencyScope`, `resolvable`, or `consumable` over
`create` when callers do not immediately need the instance.

## Inherit from a provider

`Configuration.extendsFrom` accepts a `Provider<Configuration>` as of `9.4.0`.
A lazily registered parent therefore does not need an eager `get()`:

```kotlin
configurations {
    val parent = dependencyScope("parent")
    resolvable("child") {
        extendsFrom(parent)
    }
}
```

## Merge attributes lazily

`AttributeContainer.addAllLater(source)` (`9.1.0`) lazily imports every
attribute from another container:

```kotlin
target.attributes.addAllLater(source.attributes)
```

The merge has three ordering rules:

1. Imported values override attributes already present in the destination.
2. Later changes in the source remain visible.
3. Destination values set after `addAllLater` take precedence.

## Delay publication configuration realization

Since `9.2.0`, `AdhocComponentWithVariants.addVariantsFromConfiguration(...)`
and `withVariantsFromConfiguration(...)` accept a
`Provider<ConsumableConfiguration>`. Pass the provider directly so the
configuration is realized only if its publication is published:

```kotlin
val publishedVariant = configurations.consumable("publishedVariant")

publishing {
    val component = softwareComponentFactory.adhoc("custom")
    component.addVariantsFromConfiguration(publishedVariant) {}
}
```

See
[dependencies-publishing-and-distribution.md](dependencies-publishing-and-distribution.md)
for the complete ad hoc component setup.

## Freeze collection membership without realization

Plugin authors can call `DomainObjectCollection.disallowChanges()` (`9.5.0`)
to reject later additions and removals without realizing lazily added entries:

```kotlin
val items = objects.domainObjectContainer(MyType::class)
val main = MyType("main")
items.add(main)
items.disallowChanges()
main.setFoo("bar")
```

The lock covers collection membership only. Objects already in the collection
remain mutable.
