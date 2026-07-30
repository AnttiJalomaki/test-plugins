# CLI, Wrapper, Tooling API, and Platforms

## Wrapper version selectors

On Gradle `9.0.0` or newer, `wrapper --gradle-version` accepts a major or
major/minor selector and resolves the latest matching release:

```text
./gradlew wrapper --gradle-version=9
./gradlew wrapper --gradle-version=9.1
```

Do not apply this interpretation to pre-9 values. A selector such as `8.12`
already names an exact historical release.

## Wrapper authentication, retries, and timeout

Wrapper distribution downloads accept bearer tokens through system properties
(since `9.4.0`). Bearer credentials take priority over Basic credentials.
Restrict both authentication types by host so credentials are not sent to an
unintended distribution server.

Retries are disabled by default. Enable them in
`gradle-wrapper.properties` when transient failures are expected (since
`9.5.0`):

```properties
retries=3
retryBackOffMs=1000
```

`retryBackOffMs` is the initial delay and doubles after every failed attempt.

`Wrapper.getNetworkTimeout()` is stable as of `9.6.1`; it is no longer
incubating and is covered by Gradle's backward-compatibility guarantees.

## Non-interactive and console output

Disable all prompting in unattended builds with `--non-interactive` (since
`9.6.1`):

```text
./gradlew --non-interactive build
```

The persistent equivalent is:

```properties
org.gradle.console.interactive=false
```

A non-empty `NO_COLOR` environment variable suppresses color while retaining
other styling and rich-console features such as progress bars and animations
(`9.6.1`):

```text
NO_COLOR=1 ./gradlew build
```

`--console=colored` adds highlighting without rich-console progress bars
(since `9.1.0`), which is useful for simple terminals and CI logs:

```text
./gradlew build --console=colored
```

On Windows ARM64/AArch64, the rich console is unavailable (`9.2.0`). Default
console selection and an explicit `--console=rich` both fall back to plain
output.

## Task and project diagnostics

The incubating `--task-graph` option prints a tree of requested tasks and
dependencies without executing them (since `9.1.0`):

```text
./gradlew root r2 --task-graph
```

The Project Report includes each project's physical filesystem location next
to its logical build path as of `9.1.0`.

Non-verification task failures identify the build script, settings script, or
plugin that registered the task (since `9.5.0`). The same provenance appears
in `help --task`, and it can be requested in task listings:

```text
./gradlew tasks --provenance
```

## Composite-build dry runs

`--dry-run` prevents execution-phase tasks in included builds from running as
of `9.1.0`. Tasks invoked by an included build's configuration logic can still
run during configuration, so dry-run does not imply that configuration is
side-effect-free.

## Project initialization

`init --into` creates a project in a specified target directory, creating the
directory when necessary (since `9.5.0`):

```text
gradle init --type java-application --into my-new-project
```

## Stable streamed values

Asynchronous Tooling API value streaming became stable in `8.13.0`. The
compatibility guarantee covers:

- `BuildActionExecuter.setStreamedValueListener(StreamedValueListener)`
- `StreamedValueListener`
- `BuildController.send(Object)`

Use streaming when an action should deliver intermediate values before its
final result.

## Tooling API parallelism and lightweight models

`org.gradle.tooling.parallel` controls parallel Tooling API actions
independently from `org.gradle.parallel` (since `9.4.0`). If unset, it inherits
the value of `org.gradle.parallel`:

```properties
org.gradle.tooling.parallel=true
org.gradle.parallel=false
```

`BuildEnvironment.getVersionInfo()` returns the exact `gradle --version`
output without starting a daemon, while the `Help` model exposes rendered
`gradle --help` output (`9.4.0`):

```java
String version =
    connection.getModel(BuildEnvironment.class).getVersionInfo();
String help =
    connection.getModel(Help.class).getRenderedText();
```

## Streaming TestKit output

`BuildResult.getOutputReader()` returns a `BufferedReader` as of `9.3.0`.
Process large `GradleRunner` output incrementally instead of materializing the
entire `getOutput()` string, and close the reader:

```java
try (BufferedReader reader = result.getOutputReader()) {
    boolean found = reader.lines()
        .anyMatch(line ->
            line.contains("example build message"));
}
```

## Platform support

Gradle builds can run on Windows ARM64/AArch64 devices, including Windows
virtual machines hosted on ARM hardware (since `9.2.0`). Account for the plain
console fallback described above when comparing logs across platforms.
