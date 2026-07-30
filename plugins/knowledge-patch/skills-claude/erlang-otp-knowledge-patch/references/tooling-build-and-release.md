# Tooling, build, release, and platform behavior

## Shell and tracing

Since 28.1, a remote shell can exit by closing its input stream without
terminating the remote node. The default tracer recognizes remote-shell use
and sends trace output to the remote group leader.

## Cross-reference analysis

In 29.0, `xref:analyze/2` adds the predefined analyses:

- `unsafe_function_calls`
- `undocumented_function_calls`
- `private_function_calls`

`xref` applies `ignore_xref` declarations as a post-analysis filter instead
of leaving that behavior to individual build tools.

Since 29.0.2, a BEAM file with no debug information and
`moduledoc(false)` causes `xref` to return an error rather than crash.
Callers must handle the error result.

## Testing and documentation

### Common Test map rendering

Since 28.1, Common Test prints map keys in the same order as
`maps:iterator(Map, ordered)`. Update output comparisons and tools that
consume rendered maps.

### Documentation tests

The `ct_doctest` module added in 29.0 runs shell-style examples from Erlang
module documentation and documentation files, including examples whose
expected result is failure. It can compile example modules for the test shell
and supports pluggable parsers for formats such as EDoc and AsciiDoc.

## ASN.1

`public_key` 1.19 in 28.2 adds ASN.1 support for the Public-Key Infrastructure
Certificate Management Protocol (PKICMP). When installing this as an
individual application patch, first install the OpenSSL-backed `crypto`
version shipped in 28.1.

## Archive extraction

Since 29.0, the `erl_tar` extraction option `{max_size, Size}` caps total
extracted data so an archive cannot fill the destination unchecked. Symlink
validation also accepts safe relative targets such as
`dir/link -> ../file`, which earlier releases rejected.

## Native code on Windows

Since 28.1, NIFs and linked-in drivers can be loaded while Erlang runs inside
an Erlang source tree on Windows. This enables native-code build and test
workflows in that layout.

OTP 29.0 no longer provides a 32-bit Erlang/OTP build for Windows. Move
deployment and CI jobs to a supported architecture.

## Emulator memory advice

The 28.1 emulator flag `+Mumadtn <bool>` selects `MADV_DONTNEED` instead of
`MADV_FREE`:

```text
erl +Mumadtn true
```

## Release artifacts

Since 28.4, `make release` places only runtime code in the release directory.
Generate the other artifacts separately:

```text
make release_docs
make release_tests
```

## Embedded third-party implementations

The 28.5 configure switch
`--enable-use-embedded-3pp-alternatives` forces suitable external
alternatives for affected embedded components.
`--disable-use-embedded-3pp-alternatives` selects all bundled
implementations. By default, bundled implementations are selected except
that an available operating-system `zlib` is preferred.

```text
./configure --enable-use-embedded-3pp-alternatives
```

Affected components and requirements are:

- `zstd`: external version 1.5.6 or newer;
- `zlib`: external version 1.2.5 or newer;
- Ryu with STL: a C++17 implementation;
- Tcl: glibc 2.32 `strerrorname_np()`;
- OpenSSL: no external replacement is needed because OTP uses its own MD5
  implementation.

At runtime, `erlang:system_info(embedded_3pps)` returns a map identifying the
embedded implementations in use.

## Supply-chain metadata

Since 28.3, OTP publishes per-release OpenVEX statements at
`https://erlang.org/download/vex/`, for example `otp-28.openvex.json`. They
identify vendor CVEs that do not affect Erlang/OTP so scanners can suppress
false positives. The SPDX 2.3 source SBOM links to the statement with a
security external reference.

## Crash dumps

Since 29.0, encrypted crash-dump support can be compiled in with:

```text
./configure --enable-encrypted-crash-dumps
```
