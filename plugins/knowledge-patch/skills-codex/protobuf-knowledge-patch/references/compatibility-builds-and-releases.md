# Compatibility, Builds, and Releases

Use this reference for runtime upgrades, generator selection, release planning,
CMake, Bazel, `protoc`, and plugin integration.

## Generated-code and runtime compatibility

The `release-lifecycle` guidance defines these invariants:

- Generated code must not run against a runtime older than the `protoc` and
  plugin release that produced it. Patch-level skew in this direction is also
  unsupported.
- In most languages, major-V generated code is supported from its own release
  through runtime major V+1. Runtime V+2 or later is unsupported. Older minor
  generated code can run on later runtimes in the same major.
- Regenerate on every release update. The compatibility window exists to
  permit rolling upgrades and existing-project migration.
- A security fix can require both a runtime update and regeneration despite
  the normal window.
- Loading multiple protobuf runtime majors into one process is unsupported.
- C++ and Rust require generated code and runtime to match exactly. C++ also
  provides no ABI-stability promise across minor or patch releases.
- Python generated code from 3.20.0 onward is descriptor-based and supported
  through at least runtime 8.x. A future incompatible major is expected to
  warn and then error in advance.

The poison warnings introduced in `30.0-migration` identify generated code that
still works under the rolling-upgrade policy but will fail at the next runtime
major. For example, Python 4.x generated code can run on a 5.x runtime while
warning about 6.x. Treat these warnings as a regeneration deadline.

## Shared releases and language package versions

Protobuf publishes a common `minor.point`, but language packages add their own
major. Shared release `34.1`, for example, maps to Java `4.34.1` and C#
`3.34.1`.

The provisional package boundaries from `34.0-announcement` were:

- C++ and Python move from 6.33 to 7.34.0.
- PHP and Objective-C move from 4.33 to 5.34.0.
- Java, Ruby, C#, Rust, and JRuby do not take a major bump.
- Python generated code does not change for 7.34.x, and its poison checks are
  relaxed so older generated files do not warn or fail merely because of that
  package-major change.

The `release-lifecycle` support snapshot lists:

| Product | Active line | Maintenance line or minimum gencode |
| --- | --- | --- |
| `protoc` | 35.x | 33.x and Java-specific 25.x maintenance |
| C++ | 7.35.x | 6.33.x maintenance; exact gencode match |
| C# | 3.35.x | minimum gencode 3.0.0 |
| Java | 4.35.x | 3.25.x maintenance; minimum gencode 3.0.0 |
| PHP | 5.35.x | 4.33.x maintenance; minimum gencode 4.26.0 |
| Python | 7.35.x | 6.33.x maintenance; minimum gencode 3.20.0 |
| Ruby | 4.35.x | minimum gencode 3.0.0 |

## Release and platform policy

From `release-lifecycle`:

- Updates ship quarterly, with breaking releases targeted for Q1.
- A new minor immediately ends support for the previous minor.
- After a new major, the previous major remains supported for four quarters.
  Java 3.x is an exception with a 36-month maintenance window.
- Minor and patch releases may add or deprecate `descriptor.proto` elements,
  introduce an Edition, and add or remove operating-system, language, and
  tooling support.
- Enforcing an existing support policy, such as dropping an EOL platform, is
  not treated as a breaking change requiring a language-major bump.
- On Android, the supported minimum SDK is the lower of the Google Play
  services minimum and Jetpack's default.
- JRuby is best-effort. Its target is the newest JRuby compatible with the
  minimum supported Ruby.

Edition numbers are independent of compiler and runtime versions. Edition 2023
requires `protoc` 27.0 or newer; Edition 2024 requires 32.0 or newer. Current
compilers continue to accept proto2, proto3, and both Editions.

## Language and build baselines

- `30.0-migration` raises the C++ language minimum to C++17.
- Python package 6.30 requires Python 3.9 or newer.
- `34.0-migration` raises Python to 3.10, PHP to 8.2, and Bazel to 8.
- `31.0` removes Ruby 3.0, making Ruby 3.1 the minimum at that point.
- `36.0-rc1` removes Ruby 3.1 support.
- The Objective-C breaking line begins with runtime 4.30.

## CMake dependency and distribution changes

Since `30.0-migration`, the old `protobuf_*_PROVIDER` switches are removed.
CMake first prefers installed dependencies and fetches pinned versions for
missing dependencies.

```sh
cmake . -Dprotobuf_LOCAL_DEPENDENCIES_ONLY=ON
cmake . -Dprotobuf_FORCE_FETCH_DEPENDENCIES=ON
```

Use `protobuf_LOCAL_DEPENDENCIES_ONLY=ON` to fail rather than fetch. Use
`protobuf_FORCE_FETCH_DEPENDENCIES=ON` to fetch even when an installed
dependency exists.

Other packaging changes:

- C++ CocoaPods releases are removed in `30.0-migration`; use the C++ runtime
  from the release artifacts.
- CMake installs stop shipping protoc's private generator headers in
  `34.0-announcement`. Consumers must not include those internal headers from
  the installed package.
- Protobuf's CMake tests are disabled by default in `34.0`. Source builds and
  CI that require those targets must enable them explicitly.

## Bazel dependency and Python rule migrations

The Python aliases changed in `30.0-migration`:

- `bazel/system_python.bzl` is removed. Prefer `protobuf_deps.bzl`, or use the
  moved `python/dist/system_python.bzl`.
- The internal `py_proto_library` from `protobuf.bzl` is removed. Use the
  official rule at `bazel/py_proto_library`.

`34.0-migration` requires Bazel 8. The Bazel 8 default moves dependency
management from WORKSPACE to Bzlmod, so upgrade both the tool and dependency
declarations.

`34.0` changes `@protobuf//bazel/flags:prefer_prebuilt_proto` to default to
true. Pin it explicitly if reproducible builds depend on selecting a
source-built compiler instead.

The `internal_py_proto_library` rule emits a deprecation warning in
`36.0-rc1`, ahead of Q1 2027 breaking changes. Migrate to the supported Python
proto rules before that removal.

## Bazel proto toolchains and flags

The transition described by `34.0-announcement` stops Proto rules from reading
native `--proto_toolchain_for*` and `--proto_compiler`. Temporary replacements
are:

```text
--@protobuf//bazel/flags/cc:proto_toolchain_for_cc
--@protobuf//bazel/flags/java:proto_toolchain_for_java
--@protobuf//bazel/flags/java:proto_toolchain_for_javalite
--@protobuf//bazel/flags:proto_compiler
```

The durable configuration is to enable
`--incompatible_enable_proto_toolchain_resolution` and register platform
toolchains. This is already the Bazel 9 default.

Other Proto flags move under `--@protobuf//bazel/flags`, including
`strict_proto_deps`, `strict_public_imports`,
`experimental_proto_descriptor_sets_include_source_info`, and `protocopt`.
C++ header and source suffix flags live under
`--@protobuf//bazel/flags/cc`.

In v34.0, `protocopt` is mistakenly located at
`--@protobuf//bazel/flags/cc:protocopt`. v34.1 moves it to
`--@protobuf//bazel/flags:protocopt`; the former remains a deprecated alias
only until the next breaking release.

Also update providers and Windows flags:

- Replace `ProtoInfo.transitive_imports` with `transitive_sources`.
- In `30.0-migration`, Windows Bazel builds reject MSVC and require clang-cl;
  `--define=protobuf_allow_msvc=true` is only a temporary escape hatch. The
  later `34.0-announcement` removes that flag and states that Bazel supports
  MSVC, so do not carry the escape-hatch define into v34 configuration.
- `36.0-rc1` exposes standalone plugin binaries as Bazel targets, allowing
  builds to select them without an externally arranged binary.

## Compiler invocation changes

- `35.0` rejects relative file output paths from `protoc`. Pass absolute
  output directories or paths in build and generator invocations.
- `36.0-rc1` adds `--<lang>_prefix`, allowing a compiler invocation to provide
  a language-specific prefix.
- Since `34.0-announcement`, `protoc` rejects field names longer than 2^16
  characters.

When upgrading a build, test compiler discovery, plugin selection, toolchain
resolution, generated output paths, and clean-build dependency fetching
separately.
