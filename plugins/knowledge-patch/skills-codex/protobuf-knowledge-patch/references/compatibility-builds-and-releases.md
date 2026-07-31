# Compatibility, Builds, and Releases

## Generated-code and runtime compatibility

Never run generated code against a runtime older than the `protoc` and plugin
release that produced it, including patch-version mismatches. For most
languages, major-V gencode is supported from its own release through runtime
major V+1; V+2 and later are unsupported. Older-minor gencode can run on later
runtimes in the same major. Security fixes may still require paired runtime and
regenerated-code updates, and loading multiple runtime majors into one process
is unsupported. Regenerate on every release update. (release-lifecycle)

C++ and Rust require exact generated-code/runtime release matches. C++ makes
no ABI-stability guarantee even across minor or patch releases. Python gencode
from 3.20.0 onward is descriptor-based and supported through at least runtime
8.x; a future break is expected to be preceded by poison warnings and errors.
(release-lifecycle)

The poison checks introduced around the 30.0 migration warn when an old
gencode/new-runtime pair still works under rolling upgrade policy but will fail
at the next runtime major. For example, Python 4.x gencode with runtime 5.x
warned about the move to 6.x. Python gencode itself did not change for 7.34.x,
and v34 relaxed its poison checks so old generated files do not warn or fail.
(30.0-migration, 34.0-announcement)

## Release numbering and support

The shared release is a `minor.point` number, while each runtime prepends its
own major. Release `34.1`, for example, maps to Java `4.34.1` and C# `3.34.1`.
A shared release can therefore change some language majors and not others.
(release-lifecycle)

The v34 plan moved C++ and Python to 7.34.0 after 6.33, and PHP and Objective-C
to 5.34.0 after 4.33. Java, Ruby, C#, Rust, and JRuby did not take a new major.
The announcement targeted Q1 2026. (34.0-announcement)

The supported lines recorded by the release-lifecycle guidance are:

| Component | Active | Maintenance | Minimum or matching gencode |
| --- | --- | --- | --- |
| `protoc` | 35.x | 33.x and Java-specific 25.x | — |
| C++ | 7.35.x | 6.33.x | Exact runtime match |
| C# | 3.35.x | — | 3.0.0 |
| Java | 4.35.x | 3.25.x | 3.0.0 |
| PHP | 5.35.x | 4.33.x | 4.26.0 |
| Python | 7.35.x | 6.33.x | 3.20.0 |
| Ruby | 4.35.x | — | 3.0.0 |

Active lines receive features, compatible changes, and fixes; maintenance
lines receive only critical and security fixes. Updates are quarterly, with
breaking releases targeted for Q1. A new minor ends support for its predecessor
immediately; a new major keeps the previous major supported for four quarters.
Java 3.x is the exception, with a 36-month maintenance window.
(release-lifecycle)

Minor and patch releases may add or deprecate `descriptor.proto` elements,
introduce an Edition, or add/drop OS, language, and tooling support. Enforcing
an existing support policy, including dropping an EOL platform, does not
require a language-major bump. Android supports whichever minimum SDK is lower
between Google Play services and Jetpack's default. JRuby is best-effort and
targets the latest JRuby compatible with the minimum supported Ruby.
(release-lifecycle)

## Edition and compiler numbering

Edition numbers are independent of compiler/runtime versions. Edition 2023
requires `protoc` 27.0 or later; Edition 2024 requires 32.0 or later. Current
compilers continue to accept proto2, proto3, Edition 2023, and Edition 2024.
(release-lifecycle)

The Edition 2024 announcement originally targeted protobuf 32.x in Q3 2025
and explicitly described the announced behavior as provisional.
(edition-2024-announcement)

## CMake dependency and distribution changes

The `protobuf_*_PROVIDER` switches were removed in the 30.0 migration. CMake
prefers installed dependencies and fetches pinned versions when they are
missing. Set `protobuf_LOCAL_DEPENDENCIES_ONLY=ON` to forbid fetching or
`protobuf_FORCE_FETCH_DEPENDENCIES=ON` to fetch unconditionally.
(30.0-migration)

```sh
cmake . -Dprotobuf_LOCAL_DEPENDENCIES_ONLY=ON
cmake . -Dprotobuf_FORCE_FETCH_DEPENDENCIES=ON
```

C++ CocoaPods releases were removed; consume the C++ runtime from the GitHub
release. Starting with 34.0, CMake installs omit protoc's private generator
headers, and tests are not built by default. Stop including private headers and
explicitly enable tests in source builds or CI that need the targets.
(30.0-migration, 34.0-announcement, 34.0)

## Bazel migrations

### Python rule locations

The 30.0 migration removed `bazel/system_python.bzl`; prefer
`protobuf_deps.bzl`, or use its moved `python/dist/system_python.bzl`
location. The internal `py_proto_library` in `protobuf.bzl` was removed; use
the official rule under `bazel/py_proto_library`. (30.0-migration)

### Windows compiler evolution

At 30.0, Windows Bazel builds rejected MSVC and required clang-cl. The
temporary `--define=protobuf_allow_msvc=true` suppressed the error, while CMake
continued to support MSVC. By the 34.0 changes, Bazel again supported MSVC and
the temporary allow flag was removed. Apply the state corresponding to the
pinned Protobuf release. (30.0-migration, 34.0-announcement)

### Bazel 8, Bzlmod, and toolchain resolution

Protobuf 34 drops Bazel 7; Bazel 8 is the minimum and changes the default
dependency mode from WORKSPACE to Bzlmod. Migrate dependency declarations as
part of the upgrade. (34.0-migration)

Native `--proto_toolchain_for*` and `--proto_compiler` are no longer read by
Proto rules. Short-term repository-scoped replacements are:

```text
--@protobuf//bazel/flags/cc:proto_toolchain_for_cc
--@protobuf//bazel/flags/java:proto_toolchain_for_java
--@protobuf//bazel/flags/java:proto_toolchain_for_javalite
--@protobuf//bazel/flags:proto_compiler
```

The durable migration is to enable
`--incompatible_enable_proto_toolchain_resolution` and register platform
toolchains; resolution is the Bazel 9 default. Other Proto flags move under
`--@protobuf//bazel/flags`, including `strict_proto_deps`,
`strict_public_imports`,
`experimental_proto_descriptor_sets_include_source_info`, and `protocopt`.
C++ source/header suffix flags are under `--@protobuf//bazel/flags/cc`.
(34.0-announcement)

In 34.0, `protocopt` was accidentally located at
`--@protobuf//bazel/flags/cc:protocopt`. Version 34.1 moves it to
`--@protobuf//bazel/flags:protocopt`, retaining the old location as a
deprecated alias until the next breaking release. (34.0-announcement)

Replace removed `ProtoInfo.transitive_imports` with `transitive_sources`.
At 34.0, `@protobuf//bazel/flags:prefer_prebuilt_proto` defaults to true; pin
the setting if reproducible compiler selection depends on the previous
default. (34.0-announcement, 34.0)

## Compiler and source-build behavior

Starting with 35.0, `protoc` fails writes whenever a file output path is
relative. Resolve every generator output location to an absolute path before
invocation. (35.0)

The 34.0 source-build `safe_boundary_check` mechanism was removed. Configure
boundary checking with `--//third_party/protobuf:bounds_check_mode`.
CMake no longer builds protobuf's tests by default, so explicitly request them
where test targets are part of CI. (34.0)

## Cross-runtime input hardening

At 34.0, recursion limits expanded to Java JSON `Any` nested inside `Any`, C#
JSON well-known types with deep arrays, and nested messages in Python and upb.
Expect deeply recursive inputs previously accepted by those paths to fail.
(34.0)
