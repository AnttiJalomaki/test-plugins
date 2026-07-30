# Tooling API, Problems API, and Build Logic

Use this reference when developing a Tooling API client, emitting structured
problems, writing convention plugins, resolving settings-relative files, or
updating Kotlin and Groovy DSL behavior.

## Resolve files from the settings directory

`ProjectLayout` exposes the directory containing `settings.gradle(.kts)` as of
`8.13.0`. Use it for build-wide files instead of reaching through
`rootProject`:

```kotlin
val versionFile = layout.settingsDirectory.file("version.txt")
```

## Stream custom Tooling API values

Asynchronous client streaming became stable in `8.13.0`. These APIs are
covered by Gradle's compatibility guarantees:

- `BuildActionExecuter.setStreamedValueListener(StreamedValueListener)`
- `StreamedValueListener`
- `BuildController.send(Object)`

Use them when a build action should emit values to the client before the
action's final result is available.

## Attach structured custom problem data

`ProblemSpec.additionalData(...)` accepts arbitrary typed data beginning in
`8.14.0`. The data can contain:

- Provider API properties;
- bean-style fields;
- collections;
- nested objects.

Define the producer model as an interface extending `AdditionalData`:

```java
public interface SomeData extends AdditionalData {
    Property<String> getSome();
    List<String> getNames();
    void setNames(List<String> names);
}

problem.additionalData(SomeData.class, data -> {
    data.getSome().set("some");
    data.setNames(Collections.singletonList("moreData"));
});
```

Tooling API consumers can retrieve the typed data through
`CustomAdditionalData.get(Class)` (`8.14.0`). Define a view interface that
mirrors the producer's data model, then request that view instead of parsing an
unstructured payload:

```java
SomeDataView data =
    problem.getAdditionalData().get(SomeDataView.class);
String value = data.getSome();
```

## Render Problems API entries in the console

`--warning-mode=all` displays relevant structured problem entries directly in
the console as of `9.3.0`, including their build location, and still links the
HTML Problems report:

```text
./gradlew test --warning-mode=all
```

## Configure Tooling API concurrency separately

`org.gradle.tooling.parallel` (`9.4.0`) controls parallel Tooling API actions
independently of task parallelism:

```properties
org.gradle.tooling.parallel=true
org.gradle.parallel=false
```

When `org.gradle.tooling.parallel` is unset, it inherits
`org.gradle.parallel`.

## Read version and help without a build model

`BuildEnvironment.getVersionInfo()` (`9.4.0`) returns the exact
`gradle --version` output without starting a daemon. The Tooling API `Help`
model exposes rendered `gradle --help` output:

```java
String version =
    connection.getModel(BuildEnvironment.class).getVersionInfo();
String help = connection.getModel(Help.class).getRenderedText();
```

## Stream TestKit output

`BuildResult.getOutputReader()` returns a `BufferedReader` as of `9.3.0`.
Process large `GradleRunner` output incrementally and close the reader:

```java
try (BufferedReader reader = result.getOutputReader()) {
    boolean found = reader.lines()
        .anyMatch(line -> line.contains("example build message"));
}
```

## Use compile-only plugins in precompiled scripts

Precompiled Kotlin script plugins can apply and configure plugins supplied as
`compileOnly` dependencies starting in `9.1.0`. The generated type-safe
extension accessors for those plugins are available to the script.

## Generate accessors for precompiled Settings plugins

Precompiled `*.settings.gradle.kts` plugins receive generated type-safe
accessors in `9.5.0` when their convention-plugin build applies `kotlin-dsl`:

```kotlin
// build-logic/build.gradle.kts
plugins {
    `kotlin-dsl`
}
```

The Settings plugin can then configure an applied plugin without string-based
lookups:

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

## Use registration names as plugin IDs deliberately

With `java-gradle-plugin`, a plugin registration uses its registration name as
the plugin ID unless `id` is explicitly set (`9.4.0`):

```kotlin
gradlePlugin {
    plugins {
        register("my.plugin-id") {
            implementationClass = "my.PluginClass"
        }
    }
}
```

Set `id` explicitly when the registration name is not the public plugin ID.

## Validate published plugin projects

As of `9.4.0`, applying `com.gradle.plugin-publish`, `ivy-publish`, or
`maven-publish` enables stricter plugin validation automatically. Local plugins
in `buildSrc` and included builds are exempt. Other projects can enable it:

```kotlin
tasks.validatePlugins {
    enableStricterValidation = true
}
```

`validatePlugins` provides specific errors for `@Optional` misuse in `9.6.1`.
Pair `@Optional` with an input or output annotation. Use `@Internal` alone for
an ignored property; do not combine it with `@Optional`.

## Eliminate parent-project lookup in Groovy DSL

In `9.6.1`, Groovy DSL references that resolve a missing property or method
from a parent project are deprecated. The same applies to `findProperty()`,
`property()`, and `hasProperty()` when the returned value comes from a parent.
The lookup is scheduled for removal in Gradle 10.

After removing such references, make accidental lookup fail early:

```kotlin
// settings.gradle.kts
enableFeaturePreview("NO_IMPLICIT_LOOKUP_IN_PARENT_PROJECTS")
```

## Use supported Groovy coercions for lazy properties

Groovy DSL assignment in `9.6.1` coerces a string to `Property<File>`,
`RegularFileProperty`, or `DirectoryProperty` and resolves the string relative
to the project directory.

A scalar or array can also be assigned directly to `ListProperty<T>` or
`SetProperty<T>`:

```groovy
task.workingDir = '../my-build'
task.filter.includePatterns = 'Foo'
task.filter.includePatterns = ['Foo', 'Bar'] as String[]
```
