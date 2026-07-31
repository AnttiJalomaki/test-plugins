# Compatibility and release lifecycle

## Generated code and runtime pairing (`release-lifecycle`)

Generated code must never run against a runtime older than the `protoc` and
plugin release that generated it, even when the only difference is the patch
version. For most languages, major `V` generated code is supported on runtime
major `V` and `V+1`, but runtime `V+2` or later is unsupported. Older-minor
generated code can run on later runtimes within the same major.

Security fixes may require the runtime and regenerated code to move together
despite that window. Loading multiple major versions of a runtime in the same
process is unsupported.

C++ and Rust are stricter: generated code and runtime releases must match
exactly. C++ also provides no ABI-stability guarantee between minor or patch
releases.

Python generated code from 3.20.0 onward is descriptor-based and supported
through at least runtime 8.x. If a later major eventually breaks that extended
window, poison-pill warnings and errors are expected in advance.

## Cross-version warning behavior (`30.0-migration`)

An old generated file can still work with a newer runtime under the rolling
upgrade policy while warning that the pairing will fail at the next runtime
major. For example, Python 4.x generated code works with a 5.x runtime but warns
about 6.x. Treat poison warnings as a regeneration deadline rather than an
immediate compatibility failure.

## Shared releases and language package majors (`release-lifecycle`)

The shared protobuf release uses `minor.point`, while each runtime prepends its
own package major. Release 34.1, for example, maps to Java 4.34.1 and C# 3.34.1.
A shared release can bump one language's package major without bumping another.

The planned `34.0-announcement` boundaries illustrate this: C++ and Python move
from 6.33 to 7.34.0, while PHP and Objective-C move from 4.33 to 5.34.0. Java,
Ruby, C#, Rust, and JRuby do not bump their package majors. Python generated
code does not change for 7.34.x, and poison checks are relaxed so older
generated files do not warn or fail merely because of that package-major move.

## Runtime and tool baselines

- `30.0-migration`: C++17 replaces C++14 as the minimum language level. Python
  package 6.30.x requires Python 3.9 or newer. Objective-C's first breaking
  runtime is 4.30.x, and runtime entry points for generated code older than
  3.22 are removed.
- `31.0`: the Ruby runtime requires Ruby 3.1 or newer.
- `34.0-migration`: Python requires 3.10 or newer, PHP requires 8.2 or newer,
  and Protobuf requires Bazel 8 or newer.

## Supported release lines (`release-lifecycle`)

Active lines receive features, compatible changes, and fixes; maintenance lines
receive critical and security fixes only. The stated support matrix is:

- `protoc`: 35.x active; 33.x and Java-specific 25.x in maintenance.
- C++: 7.35.x active; 6.33.x in maintenance; generated code must match exactly.
- C#: 3.35.x active; minimum generated code 3.0.0.
- Java: 4.35.x active; 3.25.x in maintenance; minimum generated code 3.0.0.
- PHP: 5.35.x active; 4.33.x in maintenance; minimum generated code 4.26.0.
- Python: 7.35.x active; 6.33.x in maintenance; minimum generated code 3.20.0.
- Ruby: 4.35.x active; minimum generated code 3.0.0.

Regenerate on every release update. Support for older generated code is an
existing-project compatibility bridge, not the preferred update workflow.

## Cadence and retirement (`release-lifecycle`)

Releases are quarterly, with breaking releases targeted for Q1. A new minor
immediately ends support for the previous minor. After a new major, the prior
major remains supported for four quarters. Java 3.x is the exception, with a
36-month maintenance window.

## Editions and compiler releases (`release-lifecycle`)

Edition numbers are independent of compiler and runtime versions. Edition 2023
requires `protoc` 27.0 or newer and Edition 2024 requires 32.0 or newer. The
latest compiler continues to accept proto2, proto3, Edition 2023, and Edition
2024 schemas.

Minor and patch releases may add or deprecate `descriptor.proto` elements,
introduce an Edition, or add and remove operating-system, language, and tooling
support. Enforcing an existing support policy, such as dropping an EOL
platform, is not treated as a language-major breaking change.

## Android and JRuby (`release-lifecycle`)

Android supports whichever minimum SDK is lower between the Google Play
services minimum and Jetpack's default. JRuby is best-effort: the target is the
latest JRuby compatible with the minimum supported Ruby version.
