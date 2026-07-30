# Testing, Reports, and Quality Tools

Use this reference when configuring `Test`, generating or consuming test
reports, integrating an external test engine, streaming TestKit output, or
validating plugins and PMD results.

## Configure custom test tasks explicitly

After the `9.0.0-upgrade`, a newly registered `Test` task does not inherit the
built-in `test` source set's classes and runtime classpath. A task that depended
on the convention silently runs no tests.

Set both inputs or create the target with JVM test suites:

```kotlin
val test by testing.suites.existing(JvmTestSuite::class)
tasks.register<Test>("otherTest") {
    testClassesDirs = files(test.map { it.sources.output.classesDirs })
    classpath = files(test.map { it.sources.runtimeClasspath })
}
```

When test sources exist and no filters apply, a test task that discovers no
tests now fails. Set `failOnNoDiscoveredTests = false` on `AbstractTestTask`
only when an empty discovery result is intentional.

## Discover definitions that are not classes

In `9.4.0`, `Test` can discover definitions handled by a custom JUnit Platform
`TestEngine` without requiring a test class. Point `testDefinitionDirs` at the
definitions:

```kotlin
tasks.named<Test>("test") {
    testDefinitionDirs.from("src/test/definitions")
}
```

This also allows Cucumber feature files to run directly without an empty suite
class or JUnit extension.

## Choose generated Kotlin test dependencies

Kotlin projects created by `init` use
`org.jetbrains.kotlin:kotlin-test` rather than `kotlin-test-junit5` beginning
in `9.1.0`. The selected test-library variant can therefore follow the
configured test runner.

## Consume richer JUnit report data

### Millisecond timestamps

JUnit XML from `Test` uses millisecond-precision test-event timestamps as of
`8.13.0`. Report consumers must accept values such as:

```xml
timestamp="2024-02-03T12:34:56.789"
```

### Assumption skip reasons

Since `8.14.0`, an assumption violation's reason appears in HTML and JUnit XML
reports. This applies to JUnit 4, JUnit Platform, and TestNG.

### Framework hierarchy and output attribution

HTML reports in `9.3.0` reflect test-framework structure:

- Nested classes appear under their enclosing class.
- A parameterized method owns its invocations.
- Suites own their classes.
- Synthetic package containers are removed.
- Standard output and error remain attached to the individual test that
  produced them.

Nested-class XML filenames remain `TEST-OuterClass$InnerClass.xml`. Suite XML
emits only class reports.

### Source-preserving aggregate reports

`TestReport` and the Test Report Aggregation Plugin no longer merge overlapping
test structures from different inputs (`9.3.0`). Each source receives its own
tab in the HTML report, keeping results attributable when subprojects reuse
suite or class names.

### Sortable HTML tables

HTML report columns are sortable as of `9.6.1`. Click once to sort, again to
reverse, and again to restore the original order.

Tests, Failures, Skipped, and Duration sort descending first. Success rate
sorts ascending first.

## Publish JUnit data and attachments

Gradle `9.4.0` captures JUnit Platform `TestReporter` key-value entries and
files, including data emitted during construction, setup, and teardown:

```java
testReporter.publishEntry("browser", "firefox");
testReporter.publishFile(
    "screenshot.svg",
    MediaType.create("image", "svg+xml"),
    file -> {}
);
```

HTML reports add Data and Attachments tabs. XML maps entries into
`<properties/>` and renders files as
`[[ATTACHMENT|/path/to/file]]`.

Build logic can consume structured metadata directly rather than parsing
standard output:

```kotlin
tasks.named<Test>("test") {
    addTestMetadataListener(LoggingListener())
}
```

`Test.addTestMetadataListener(TestMetadataListener)` receives the metadata
events.

## Report tests run outside built-in infrastructure

Plugin and platform authors can inject `TestEventReporterFactory` (`8.13.0`)
to write Gradle-style binary results and HTML reports for externally executed
tests. A reporter supports:

- `started(...)`, `succeeded(...)`, and the corresponding lifecycle;
- timestamped `metadata(...)`;
- nested `reportTestGroup(...)` groups;
- `reportTest(...)` events.

```java
try (TestEventReporter test =
        testEventReporterFactory.createTestEventReporter(
            "custom-test",
            binaryResultsDirectory,
            htmlReportDirectory)) {
    test.started(Instant.now());
    test.metadata(Instant.now(), "engine", "custom");
    test.succeeded(Instant.now());
}
```

Close the reporter so the report is finalized.

## Stream TestKit output

`BuildResult.getOutputReader()` returns a `BufferedReader` in `9.3.0`. Use it
to process high-volume `GradleRunner` output incrementally instead of
materializing the full `getOutput()` string:

```java
try (BufferedReader reader = result.getOutputReader()) {
    boolean found = reader.lines()
        .anyMatch(line -> line.contains("example build message"));
}
```

Always close the reader.

## Surface structured problems during a test run

`--warning-mode=all` renders relevant structured Problems API entries in the
console as of `9.3.0`, including their build location, while retaining the
link to the HTML Problems report:

```text
./gradlew test --warning-mode=all
```

## Configure additional PMD formats

Individual `Pmd` tasks can emit CSV, Code Climate, and SARIF reports starting
in `9.4.0`. They are disabled by default and must be configured on the task,
not the `pmd` extension:

```kotlin
tasks.pmdMain {
    reports {
        csv.required = true
        codeClimate.required = true
        sarif.required = true
    }
}
```

## Apply stricter plugin validation

Applying `com.gradle.plugin-publish`, `ivy-publish`, or `maven-publish`
automatically enables stricter plugin validation as of `9.4.0`. Local plugins
in `buildSrc` and included builds are exempt. Other plugin projects can opt in:

```kotlin
tasks.validatePlugins {
    enableStricterValidation = true
}
```

In `9.6.1`, `validatePlugins` reports targeted errors for two forms of
`@Optional` misuse:

- `@Optional` without an input or output annotation: add the appropriate
  input/output annotation.
- `@Optional` together with `@Internal`: use only `@Internal` for an ignored
  property.

Do not combine `@Internal` with `@Optional`.

## Account for Gradle 9 quality-tool defaults

The `9.0.0-upgrade` changes the defaults to Checkstyle 10.24.0, CodeNarc
3.6.0, PMD 7.13.0, JUnit Jupiter 5.12.2, TestNG 7.11.0, and Spock 2.3.
Gradle's JGit is 7.2.1 and can use an SSH agent.
