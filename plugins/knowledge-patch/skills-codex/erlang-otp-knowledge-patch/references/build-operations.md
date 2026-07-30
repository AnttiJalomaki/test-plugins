# Build and Operations

## PCRE2 migration and compiled expressions

The `re` module uses PCRE2 starting in OTP 28.0. Pattern validation is
stricter: invalid escapes including `\M`, `\i`, `\B`, or `\8` can now raise
`badarg`. Unicode property results can change with newer property data, and
branch-reset groups can alter `re:split/3` output. Recompile and retest both
accepted and rejected patterns during an upgrade.

Do not persist or transfer the internal value returned by `re:compile/2`; it
is not reusable across Erlang nodes or OTP versions. OTP 28.1 adds supported
export and import operations for compiled regular expressions so they can be
transferred safely between node instances. Use that path instead of treating
the internal value as portable.

## Release artifacts

In OTP 28.4, `make release` puts only runtime code in the release directory.
Documentation and tests have separate targets:

```text
make release
make release_docs
make release_tests
```

Update packaging jobs that assumed the main release target also produced docs
or test artifacts.

## Embedded third-party implementations

OTP 28.5 adds explicit configuration for the implementations used by affected
embedded components:

```text
./configure --enable-use-embedded-3pp-alternatives
```

`--enable-use-embedded-3pp-alternatives` forces suitable external
alternatives, while `--disable-use-embedded-3pp-alternatives` selects all
bundled implementations. By default, bundled implementations are selected
except that an available operating-system `zlib` is preferred.

The affected components are `zstd`, `zlib`, Ryu with STL, OpenSSL, and Tcl.
External alternatives have these prerequisites:

- `zstd` 1.5.6 or newer;
- `zlib` 1.2.5 or newer;
- C++17 for Ryu; and
- glibc 2.32 `strerrorname_np()` for Tcl.

OpenSSL needs no external replacement because OTP uses its own MD5
implementation. At runtime, inspect `erlang:system_info(embedded_3pps)` for a
map describing the embedded implementations in use.

## Memory advice selection

The emulator flag `+Mumadtn <bool>` in OTP 28.1 selects `MADV_DONTNEED`
instead of `MADV_FREE`:

```text
erl +Mumadtn true
```

Make this an explicit runtime choice where operating-system memory behavior is
part of deployment tuning.

## Windows platform changes

OTP 28.1 permits NIFs and linked-in drivers to load on Windows while Erlang is
running in an Erlang source tree. This enables native-code builds and tests in
that layout.

OTP 29.0 ends production of 32-bit Erlang/OTP builds for Windows. Move build,
packaging, and deployment pipelines to a supported architecture before
upgrading.

## Vulnerability metadata

Starting with OTP 28.3, each release has an OpenVEX statement under
`https://erlang.org/download/vex/`, for example `otp-28.openvex.json`. These
statements identify reported CVEs that do not affect Erlang/OTP so scanners can
avoid false positives. The SPDX 2.3 source SBOM links to the OpenVEX document
through a security external reference.

## Crash-dump builds

OTP 29.0 can be built with encrypted crash-dump support by passing
`--enable-encrypted-crash-dumps` to `configure`. Coordinate this build choice
with the operational process for retaining and reading crash dumps.
