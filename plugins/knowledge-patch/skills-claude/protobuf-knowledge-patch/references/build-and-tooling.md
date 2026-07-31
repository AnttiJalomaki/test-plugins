# Build and tooling

## Dependency fetching in CMake (`30.0-migration`)

The `protobuf_*_PROVIDER` switches are removed. CMake first uses installed
dependencies and fetches pinned versions for anything missing. Set
`protobuf_LOCAL_DEPENDENCIES_ONLY=ON` to prohibit network fetching, or
`protobuf_FORCE_FETCH_DEPENDENCIES=ON` to fetch even when a dependency is
installed.

```sh
cmake . -Dprotobuf_LOCAL_DEPENDENCIES_ONLY=ON
cmake . -Dprotobuf_FORCE_FETCH_DEPENDENCIES=ON
```

Protobuf's CMake tests are disabled by default as of `34.0`. Source builds and
CI that need test targets must enable them explicitly.

## CMake packaging and distribution

The C++ CocoaPods distribution was removed in `30.0-migration`; obtain the C++
runtime from the GitHub release. Protoc's private generator headers are also no
longer installed by CMake as planned in `34.0-announcement`. Those headers were
never public, so downstream generators must stop expecting an installed package
to provide them.

## Python Bazel rule moves (`30.0-migration`)

The `bazel/system_python.bzl` alias is gone. Prefer `protobuf_deps.bzl`, or use
the moved file at `python/dist/system_python.bzl`. The internal
`py_proto_library` in `protobuf.bzl` was removed; use the official rule under
`bazel/py_proto_library`.

## Windows Bazel toolchains

At `30.0-migration`, Windows Bazel builds rejected MSVC and required clang-cl.
`--define=protobuf_allow_msvc=true` was only a temporary escape hatch; CMake
continued to support MSVC.

The later `34.0-announcement` restores/continues Bazel MSVC support and removes
that temporary `--define`. Remove the flag and test the real Windows toolchain
instead of carrying the transition override.

## Proto toolchain resolution (`34.0-announcement`)

Native Bazel `--proto_toolchain_for*` and `--proto_compiler` flags are no longer
read by Proto rules. Their short-term replacements are:

```text
--@protobuf//bazel/flags/cc:proto_toolchain_for_cc
--@protobuf//bazel/flags/java:proto_toolchain_for_java
--@protobuf//bazel/flags/java:proto_toolchain_for_javalite
--@protobuf//bazel/flags:proto_compiler
```

The durable migration is to enable
`--incompatible_enable_proto_toolchain_resolution` and register ordinary
platform toolchains. It is already the Bazel 9 default.

Other Proto flags move below `--@protobuf//bazel/flags`, including
`strict_proto_deps`, `strict_public_imports`,
`experimental_proto_descriptor_sets_include_source_info`, and `protocopt`.
C++ header/source suffix flags live below `--@protobuf//bazel/flags/cc`.

In 34.0, `protocopt` is accidentally located at
`--@protobuf//bazel/flags/cc:protocopt`. Version 34.1 moves it to
`--@protobuf//bazel/flags:protocopt`; the old location remains a deprecated alias
only until the next breaking release.

`ProtoInfo.transitive_imports` is removed. Read `transitive_sources` instead.

## Bazel 8 and Bzlmod (`34.0-migration`)

Protobuf 34 requires Bazel 8 or newer. Bazel 8 also switches the default
dependency mode from WORKSPACE to Bzlmod, so upgrade both the build version and
the dependency declaration strategy.

## Compiler and boundary-check selection (`34.0`)

`@protobuf//bazel/flags:prefer_prebuilt_proto` now defaults to true. Pin it when
reproducible builds depend on compiling `protoc` from source or on another
selection policy.

The former `safe_boundary_check` mechanism is removed. Configure source-build
boundary checking with:

```text
--//third_party/protobuf:bounds_check_mode
```

## Option-only imports in Bazel

Edition 2024 `import option` dependencies belong in `option_deps`, not `deps`.
The attribute requires Bazel 8 or newer.

```build
proto_library(
  name = "foo",
  srcs = ["foo.proto"],
  option_deps = [":custom_option"],
)
```

## Absolute `protoc` output paths (`35.0`)

`protoc` now rejects a file write when the resulting output path is relative.
Resolve every generator output directory to an absolute path before invoking
the compiler, including paths assembled by wrappers or custom plugins.
