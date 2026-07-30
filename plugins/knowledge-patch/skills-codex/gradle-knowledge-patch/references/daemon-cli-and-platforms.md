# Daemon, CLI, Wrapper, and Platforms

Use this reference for JVM selection, daemon connectivity, Wrapper downloads,
unattended execution, console output, platform support, and command-line
diagnostics.

## Select and provision the daemon JVM

### Meet the Gradle 9 runtime floor

The daemon requires JVM 17 or newer after the `9.0.0-upgrade`. Compilation,
tests, and workers may still target older JVMs through toolchains. Wrapper and
command-line launchers, Tooling API clients, and TestKit may run on JVM 8, but
the launcher must still locate a JVM 17+ daemon.

Java toolchain auto-detection also considers the JDK referenced by `JAVA_HOME`
as of `9.0.0`, aligning command-line and IDE discovery.

### Auto-provision a matching daemon JVM

Since `8.13.0`, Gradle can download a JDK when no installation matches the
Daemon JVM criteria. Apply Foojay resolver convention plugin 0.9.0 or provide a
custom resolver:

```kotlin
plugins {
    id("org.gradle.toolchains.foojay-resolver-convention") version "0.9.0"
}
```

Then generate the criteria file:

```text
./gradlew updateDaemonJvm --jvm-version=17 --jvm-vendor=adoptium
```

`gradle/gradle-daemon-jvm.properties` records the requested vendor and version
plus per-platform download URLs. Daemon toolchains are stable and covered by
compatibility guarantees starting in `9.2.0`; using the criteria no longer
emits an incubation warning.

### Require Native Image

Java and Daemon JVM toolchain selection can require a JDK that provides
GraalVM Native Image (`8.14.0`):

```kotlin
java {
    toolchain {
        languageVersion = JavaLanguageVersion.of(21)
        nativeImageCapable = true
    }
}
```

### Account for newer Java releases

Gradle can run its daemon on Java 25 and use Java 25 toolchains as of `9.1.0`.
Tooling API clients must enable native access at startup because the API uses
JNI. Java 26 daemon and toolchain support follows in `9.4.0`. In either case,
verify third-party tool compatibility separately.

## Control daemon networking and retention

Set `GRADLE_DAEMON_BIND_ADDRESS` (`9.5.0`) to bypass address auto-detection and
choose the address used for client-daemon and cross-daemon communication:

```text
GRADLE_DAEMON_BIND_ADDRESS=192.168.1.10 ./gradlew build
```

This is useful on multi-interface hosts and unusual network setups.

Daemon logs older than 14 days are automatically removed when the daemon shuts
down starting in `9.4.0`.

## Run on supported native platforms

Gradle supports Windows ARM64/AArch64 hosts as of `9.2.0`, including Windows
virtual machines hosted on ARM. The rich console is unavailable there, so both
automatic console selection and an explicit `--console=rich` fall back to
plain output.

## Manage Wrapper versions and downloads

### Resolve partial version selectors

Gradle `9.0.0` and newer lets `wrapper --gradle-version` accept a major or
major/minor selector and resolve the latest matching release:

```text
./gradlew wrapper --gradle-version=9
./gradlew wrapper --gradle-version=9.1
```

Do not interpret a pre-9 value the same way. For example, `8.12` is already an
exact historical version.

Gradle 9 and later uses `MAJOR.MINOR.PATCH` versions. Older releases and
backports are not renamed. Internal and `@Incubating` APIs remain outside the
public Semantic Versioning guarantee and may change in a minor release.

### Authenticate downloads with bearer tokens

Wrapper distribution downloads accept bearer tokens through system properties
as of `9.4.0`. Bearer credentials take precedence over Basic credentials.
Restrict both authentication types per host to avoid sending credentials to
unintended servers.

### Retry failed downloads

Retries are disabled by default. Enable them in
`gradle-wrapper.properties` (`9.5.0`):

```properties
retries=3
retryBackOffMs=1000
```

`retryBackOffMs` is the initial delay and doubles after each failed attempt.

### Configure network timeout through the API

`Wrapper.getNetworkTimeout()` is stable, no longer incubating, and covered by
backward-compatibility guarantees as of `9.6.1`.

### Verify distribution authenticity

Since `9.3.0`, each Gradle distribution ZIP has an ASCII-armored `.asc`
signature alongside its `.sha256` checksum. Verify the signature when
authenticity matters; the checksum alone detects corruption but does not
authenticate the publisher.

## Make console behavior explicit

`--console=colored` (`9.1.0`) adds color without rich-console features such as
progress bars, making it suitable for simple terminals and readable CI logs:

```text
./gradlew build --console=colored
```

A non-empty `NO_COLOR` environment variable (`9.6.1`) suppresses color while
retaining other styling and rich-console features:

```text
NO_COLOR=1 ./gradlew build
```

For unattended builds, disable all prompts with `--non-interactive` or the
persistent equivalent (`9.6.1`):

```text
./gradlew --non-interactive build
```

```properties
org.gradle.console.interactive=false
```

## Inspect tasks and project layout

The incubating `--task-graph` option (`9.1.0`) prints the requested tasks and
their dependency tree without executing them:

```text
./gradlew root r2 --task-graph
```

`--dry-run` also suppresses execution-phase tasks in included builds as of
`9.1.0`. Tasks invoked by an included build's configuration logic can still
run during configuration.

Project Report includes each project's physical filesystem location alongside
its logical build path starting in `9.1.0`. This makes non-standard layouts
visible.

Task diagnostics gained provenance in `9.5.0`. Non-verification failures say
which build script, settings script, or plugin registered the task;
`help --task` includes the same information. Request it in task listings with:

```text
./gradlew tasks --provenance
```

## Initialize and scan builds from the CLI

`init --into` (`9.5.0`) creates a project in the selected directory and creates
the directory if needed:

```text
gradle init --type java-application --into my-new-project
```

Publish a Build Scan to a Develocity server without project configuration by
using `--develocity-url` (`9.5.0`):

```text
./gradlew --develocity-url https://develocity.example.com build
```
