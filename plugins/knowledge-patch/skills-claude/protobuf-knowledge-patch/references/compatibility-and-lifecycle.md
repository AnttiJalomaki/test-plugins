# Compatibility and release lifecycle

Source batches represented here: 30.0-migration, 34.0-announcement,
release-lifecycle.

## Generated code and runtime pairing

Generated code must not run against a runtime older than the `protoc` and
plugin release that produced it, even when the mismatch is only at patch level.
For most languages, major V gencode is supported from its own release through
runtime major V+1; V+2 or later is unsupported. Older minor gencode can run on
later runtimes in the same major.

Security fixes can require a paired runtime upgrade and regeneration despite
that compatibility window. Loading multiple major runtime versions into one
process is unsupported.

C++ and Rust are stricter: generated code and runtime releases must match
exactly. C++ also makes no ABI-stability guarantee across minor or patch
releases.

Python gencode from 3.20.0 onward is descriptor-based and is supported through
at least runtime 8.x. Any future break in that extended window is expected to
be preceded by poison-pill warnings and errors.

Old gencode paired with a newer runtime can warn one major before failure under
the rolling-upgrade policy. For example, Python 4.x gencode with a 5.x runtime
warned about the v6 incompatibility. The planned v34 Python 7.34.x line does
not change gencode, and its poison checks are relaxed so older generated files
do not warn or fail solely for that transition.

Regenerate generated code on every release update; support for older output is
an existing-project compatibility allowance, not a reason to keep stale
artifacts.

## Shared releases and language package majors

Protobuf publishes a shared `minor.point` release, while each runtime prepends
its own major. For example, release 34.1 maps to Java 4.34.1 and C# 3.34.1.
A shared release can raise one language's major without raising another's.

At the planned v34 boundary, C++ and Python move from 6.33 to 7.34, and PHP and
Objective-C move from 4.33 to 5.34. Java, Ruby, C#, Rust, and JRuby do not take
a major bump.

## Supported release lines

The support snapshot in the release-lifecycle batch is:

- `protoc`: 35.x active; 33.x and Java-specific 25.x in maintenance.
- C++: 7.35.x active; 6.33.x in maintenance; exact gencode/runtime match.
- C#: 3.35.x active; minimum gencode 3.0.0.
- Java: 4.35.x active; 3.25.x in maintenance; minimum gencode 3.0.0.
- PHP: 5.35.x active; 4.33.x in maintenance; minimum gencode 4.26.0.
- Python: 7.35.x active; 6.33.x in maintenance; minimum gencode 3.20.0.
- Ruby: 4.35.x active; minimum gencode 3.0.0.

Active lines receive features, compatible changes, and fixes. Maintenance
lines receive critical and security fixes only.

## Cadence and retirement

Updates are quarterly, with breaking releases targeted for Q1. A new minor
immediately ends support for the previous minor. A new major keeps the
previous major supported for four more quarters; Java 3.x is an exception with
a 36-month maintenance window.

Minor and patch releases may add or deprecate `descriptor.proto` elements,
introduce an Edition, or add and drop operating-system, language, and tooling
support. Enforcing an existing policy, such as dropping an end-of-life
platform, is not classified as a breaking change and need not wait for a
language-major bump.

## Editions are an independent version axis

Edition numbers do not track compiler or runtime versions. Edition 2023 needs
`protoc` 27.0 or newer; Edition 2024 needs 32.0 or newer. Current compilers
continue to accept proto2, proto3, and both Editions.

## Platform support details

On Android, the supported minimum SDK is the lower of the Google Play services
minimum and Jetpack's default. JRuby remains best-effort, targeting the newest
JRuby compatible with the minimum supported Ruby version.
