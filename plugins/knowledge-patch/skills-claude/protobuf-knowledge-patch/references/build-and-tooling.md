# Build systems, compiler, and distribution

Source batches represented here: 30.0-migration, 34.0-announcement,
34.0-migration, 34.0, 35.0, 36.0-rc1.

## CMake dependency and target changes

The v30 migration removes the old `protobuf_*_PROVIDER` switches. CMake now
prefers installed dependencies and fetches pinned versions when missing:

```sh
cmake . -Dprotobuf_LOCAL_DEPENDENCIES_ONLY=ON
cmake . -Dprotobuf_FORCE_FETCH_DEPENDENCIES=ON
```

`protobuf_LOCAL_DEPENDENCIES_ONLY=ON` forbids fetching;
`protobuf_FORCE_FETCH_DEPENDENCIES=ON` always fetches.

Private protoc generator headers are no longer installed by CMake in the v34
line. Consumers must stop including non-public generator headers from an
installed protobuf package.

Protobuf's tests are disabled by default in v34 CMake builds. Source builds and
CI that require test targets must enable them explicitly.

## Bazel migration

The v30 Python migration removes `bazel/system_python.bzl`. Prefer
`protobuf_deps.bzl`, or use its moved path at
`python/dist/system_python.bzl`. The internal `py_proto_library` in
`protobuf.bzl` is also removed; use the official rule under
`bazel/py_proto_library`.

Protobuf v34 requires Bazel 8. Bzlmod becomes Bazel's default dependency mode,
so migrate both the tool version and any WORKSPACE-only dependency setup.

Native `--proto_toolchain_for*` and `--proto_compiler` flags are no longer read
by Proto rules. Short-term replacements are:

```text
--@protobuf//bazel/flags/cc:proto_toolchain_for_cc
--@protobuf//bazel/flags/java:proto_toolchain_for_java
--@protobuf//bazel/flags/java:proto_toolchain_for_javalite
--@protobuf//bazel/flags:proto_compiler
```

The durable migration is to enable
`--incompatible_enable_proto_toolchain_resolution`, already the Bazel 9
default, and register ordinary platform toolchains.

Other flags move under `--@protobuf//bazel/flags`, including
`strict_proto_deps`, `strict_public_imports`,
`experimental_proto_descriptor_sets_include_source_info`, and `protocopt`.
C++ header/source suffix flags live under `--@protobuf//bazel/flags/cc`.
In v34.0, `protocopt` is mistakenly located at
`--@protobuf//bazel/flags/cc:protocopt`; v34.1 moves it to
`--@protobuf//bazel/flags:protocopt` and keeps the old location as a
deprecated alias until the next breaking release.

Replace `ProtoInfo.transitive_imports` with `transitive_sources`.
`@protobuf//bazel/flags:prefer_prebuilt_proto` defaults to true in v34; pin it
when reproducible toolchain selection requires the old default.

The old `safe_boundary_check` mechanism is removed. Configure source-build
boundary checking with:

```text
--//third_party/protobuf:bounds_check_mode
```

`internal_py_proto_library` emits a deprecation warning in 36.0-rc1 ahead of
Q1 2027 breaking changes. Move to the supported Python proto rule.

Standalone plugin binaries now have exposed Bazel targets in 36.0-rc1, so
build rules can select those targets without arranging external binaries.

## Bazel on Windows

The v30 migration rejects MSVC in Windows Bazel builds and directs builds to a
clang-cl toolchain. `--define=protobuf_allow_msvc=true` temporarily suppresses
that error; MSVC remains available through CMake.

The later v34 announcement says Bazel supports MSVC again, while removing the
temporary `--define=protobuf_allow_msvc` flag. Scope the migration to the
protobuf release: do not preserve the temporary flag when moving to v34.

## `protoc` behavior

In v35, `protoc` rejects any relative file output path. Generators and build
rules must provide absolute output locations.

The compiler rejects field names longer than 2^16 characters in the v34
boundary instead of accepting arbitrary length.

In 36.0-rc1, a reserved field-number declaration may not contain `INT_MAX`.
Replace sentinel-style reservations with a valid range.

`protoc` adds `--<lang>_prefix` for supplying a language-specific prefix at
invocation time.

Generic service options `py_generic_service`, `cc_generic_service`, and
`java_generic_service` are deprecated and now warn. Plan their removal rather
than suppressing the warnings.

## Packaging and distribution

The C++ CocoaPods distribution is removed in the v30 migration. Obtain the C++
runtime from the GitHub release instead.

The C# `Google.Protobuf.Tools` NuGet package includes an `include` directory
containing the well-known-type `.proto` files in v35. Packaged compiler
invocations can resolve those imports directly from the package.
