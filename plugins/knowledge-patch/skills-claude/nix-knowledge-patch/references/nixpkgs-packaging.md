# Nixpkgs Packaging

Use this reference when maintaining Nixpkgs expressions, overlays, build
helpers, hooks, language packages, platform support, or Nixpkgs library calls.

## Core derivation and environment contracts

### Strict derivation `env` values

Since nixos-25.11, `stdenv.mkDerivation` and related builders require `env` to
be an attribute set. To create a variable literally named `env`, use:

```nix
stdenv.mkDerivation {
  env.env = "value";
}
```

### Structured attributes with debug reference checks

Since nixos-25.11, a derivation combining `separateDebugInfo` with any of
`allowedReferences`, `allowedRequisites`, `disallowedReferences`, or
`disallowedRequisites` must set `__structuredAttrs = true`. Those reference
allow- and deny-lists do not apply to the generated `debug` output.

### Fixed-point `buildEnv`

Since nixos-25.11, `buildEnv` takes `finalAttrs: { ... }` fixed-point
arguments. Its custom result `.override` is deprecated. Put extra
`stdenv.mkDerivation` arguments under `derivationArgs`; direct
`nativeBuildInputs` and `buildInputs` survive only through compatibility.

### Structured-only `buildEnv`

Since nixos-26.05, `buildEnv` uses only structured attributes
(`__structuredAttrs = true`), completing the fixed-point and `derivationArgs`
migration.

### `meta.mainProgram` participates in builds

Since nixos-25.11, `meta.mainProgram` determines `NIX_MAIN_PROGRAM`, so changing
it can rebuild a package. `versionCheckHook` may fail when `pname` differs from
the chosen main program instead of silently checking the `pname` executable.

### Literal `requireFile` strings

Since nixos-26.05, `requireFile` treats `message` and `url` as literal strings,
not Bash here-document input. Forms such as `$PWD` do not expand and need no
shell-oriented escaping.

## Output validation, hardening, and filesystems

### Broken-symlink checks

Since nixos-25.05, the `no-broken-symlinks` hook rejects dangling and reflexive
symlinks in outputs. Set `dontCheckForBrokenSymlinks = true` only when the
derivation intentionally requires them.

### Stricter symlink checks and PIE handling

Since nixos-25.11, the symlink hook also rejects output links into `$TMPDIR`,
normally `/build`. The `pie` hardening flag was removed because toolchains
enable PIE by default. A package that cannot use PIE must pass `-no-pie` in
`CFLAGS`.

### glibc executable-stack policy

In nixos-26.05, glibc 2.42 no longer makes the stack executable solely because
a loaded shared library requests it. Rebuild with
`env.NIX_LDFLAGS = "-z,noexecstack"` or clear a mistaken marker with
`patchelf --clear-execstack`. Use
`GLIBC_TUNABLES=glibc.rtld.execstack=2` only per process for code that truly
needs an executable stack.

### XFS feature compatibility

In nixos-26.05, xfsprogs 6.18 enables parent pointers and exchange-range by
default. Use a 6.18-or-newer kernel for filesystems created with those features;
GRUB 2 may not boot from them.

## Language ecosystem builders

### Stable Rust vendoring hashes

In nixos-25.05, Cargo 1.84 invalidated existing `cargoHash` values.
`rustPlatform.fetchCargoVendor` replaces the Cargo-format-dependent
`fetchCargoTarball`. `buildRustPackage` no longer accepts deprecated
`cargoSha256`; use and regenerate `cargoHash`.

### Go builder argument migration

In nixos-25.05, removed `buildGoPackage` becomes `buildGoModule`. Put
`CGO_ENABLED` at `env.CGO_ENABLED`; direct `GOOS` and `GOARCH` arguments error.
Use `goSum` and self-referencing `finalAttrs` inputs when required.

### Python hook flag representation

In nixos-25.05, Python hook flags follow `stdenv.mkDerivation`: space-separated
variables without structured attributes and Bash arrays with them, without
Bash-evaluating the values. Use `pytestFlags` and `unittestFlags`; the
`*FlagsArray` forms are compatibility-only.

### Explicit modern Python build formats

Since nixos-25.11, `buildPythonPackage` and `buildPythonApplication` require an
explicit format. For modern setuptools, set `pyproject = true` and
`build-system = [ setuptools ]`. Passing `stdenv` directly is deprecated;
override the helper instead:

```nix
(buildPythonPackage.override { stdenv = customStdenv; }) {
  pyproject = true;
  build-system = [ setuptools ];
}
```

### Default compiler and language runtimes

In nixos-26.05, defaults move from GCC 14 to GCC 15, Node.js 22 LTS to 24 LTS,
and Ruby 3.3 to 3.4. Unpinned package expressions can inherit upstream
incompatibilities from these transitions.

### Node package-set and Yarn 1 migrations

In nixos-26.05, the `nodePackages` set and `node2nix` are removed; use
top-level packages or current JavaScript packaging helpers. Removed
`yarn2nix`, `mkYarnPackage`, and related Yarn 1 tools become
`yarnBuildHook`, `yarnConfigHook`, and `yarnInstallHook`.

### Node wrapper outputs

In nixos-26.05, `nodejs_latest` names Node 26. `nodejs` is a non-overridable
wrapper around `nodejs-slim` plus npm and Corepack outputs. Override
`nodejs-slim` directly and include `nodejs-slim.corepack` when using the slim
package.

### Pnpm dependency fetchers

In nixos-26.05, top-level `fetchPnpmDeps` and `pnpmConfigHook` replace
`pnpm.fetchDeps` and `pnpm.configHook`. Fetcher versions 1 and 2 are deprecated;
regenerate pnpm hashes with version 3. Npm workspaces can set
`npmDepsFetcherVersion = 2` on `buildNpmPackage` for packument caching.

### Neovim configuration delivery

In nixos-26.05, Nixpkgs disables Neovim Python 3 and Ruby providers by default.
Lua dependencies are recorded in generated `init.lua`, not `LUA_PATH` wrapper
arguments. Commands needing those dependencies must run after initialization
with `-c`; `wrapRc = false` users must load the generated init file themselves.

## Fetchers, substitution, and generated caches

### `replaceVars` supersedes substitution helpers

Since nixos-25.05, `substituteAll` and `substituteAllFiles` are deprecated in
favor of `replaceVars` and scheduled for removal.

### Binary-cache compression default

Since nixos-25.05, `mkBinaryCache` creates zstd-compressed caches by default.
Pass `compression = "xz";` for the former format.

### Fetcher-wide policy and source-subdirectory controls

Since nixos-25.11, Nixpkgs configuration can set:

- `rewriteURL` and `hashedMirrors` for `fetchurl`;
- `gitConfig` or `gitConfigFile` for all `fetchgit`;
- `npmRegistryOverrides` or `npmRegistryOverridesString` for all
  `fetchNpmDeps`.

Individual `fetchgit` calls accept `gitConfigFile` and `rootDir`;
`fetchNpmDeps` accepts `npmRegistryOverridesString`.

## Libraries, formats, and module types

### Recursive package scopes

Since nixos-25.05, `lib.packagesFromDirectoryRecursive` rejects unknown
arguments and can construct nested scopes matching a directory tree.

### Instantiated systemd format

Since nixos-25.11, direct use of `pkgs.formats.systemd` is deprecated.
Instantiate it:

```nix
systemdFormat = pkgs.formats.systemd { };
```

### Nixpkgs library removals

Since nixos-25.11, use these direct replacements:

| Removed | Replacement |
| --- | --- |
| `cartesianProductOfSets` | `lib.attrsets.cartesianProduct` |
| `zipWithNames`, `zip` | `zipAttrsWithNames`, `zipAttrsWith` |
| `literalExample` | `literalExpression` or `literalMD` |
| `mapAttrsFlatten` | `lib.attrsets.mapAttrsToList` |
| `lib.modules.defaultPriority` | `defaultOverridePriority` |
| `mkPackageOptionMD` | `mkPackageOption` |
| `replaceChars` | `replaceStrings` |
| `lib.sources.path*` | matching `lib.filesystem` helpers |
| `lib.types.string` | a concrete type such as `lib.types.str` |

`lib.strings.isCoercibleToString` splits into `isStringLike` and broader
`isConvertibleWithToString`.

### Correct type checking inside `freeformType`

Since nixos-25.11, `types.either` no longer silently accepts mismatches when
used as a `freeformType`; this also affects `oneOf`, `number`, and `numbers.*`.
Module authors commonly need `attrsOf` around the union so the free-form value
has the intended shape.

### Replacement command-line rendering APIs

Since nixos-25.11, `lib.cli.toGNUCommandLine` and `toGNUCommandLineShell` are
deprecated. Use `toCommandLine`, `toCommandLineShell`, `toCommandLineGNU`, or
`toCommandLineShellGNU`; choose GNU variants only for GNU rendering.

### Fixed-point-free Nixpkgs configuration

In nixos-26.05, Nixpkgs configuration functions receive `lib` directly along
with `pkgs`, avoiding package fixed-point recursion when only library values
are needed.

```nix
{ lib, ... }: {
  allowlistedLicenses = [ lib.licenses.nasa13 ];
}
```

## Package scopes, platforms, and outputs

### Mesa packaging inputs

In nixos-25.05, applications linked to different Mesa versions can coexist.
Packages requiring GBM or DRI metadata should depend on `libgbm` or
`dri-pkgconfig-stub`, respectively, instead of Mesa itself.

### Split PostgreSQL development tools and extensions

In nixos-25.05, `postgresql` and `libpq` no longer include `pg_config`; add
`postgresql.pg_config` or `libpq.pg_config` to `nativeBuildInputs`.
Select PL/Python, PL/Perl, and PL/Tcl with `postgresql.withPackages` instead of
support overrides.

```nix
nativeBuildInputs = [ postgresql.pg_config ];
postgresql.withPackages (ps: [ ps.plpython3 ])
```

### Per-font Nerd Fonts packages

In nixos-25.05, monolithic `nerdfonts` splits into packages below
`nerd-fonts`. Installed files gain a per-font directory below
`share/fonts/{opentype,truetype}/NerdFonts/`; migrate both package attributes
and paths.

### Stable `nixfmt` package name

Since nixos-25.11, use `pkgs.nixfmt`. `pkgs.nixfmt-rfc-style` is deprecated,
while the former formatter remains temporarily as `pkgs.nixfmt-classic`.

### Darwin platform floor and system libc++

Nixpkgs nixos-25.11 requires macOS 14.0 or newer and defaults to SDK 14.4.
Darwin builds use the system libc++; packages needing newer C++ library
features must raise their deployment target.

### Nixpkgs evaluation and kernel-output compatibility

Evaluating Nixpkgs nixos-25.11 requires Nix 2.18 or newer. Linux packages move
all in-tree kernel modules to a separate `modules` output; consumers must not
assume they remain in the primary output.

### Final Intel Darwin release

nixos-26.05 is the final Nixpkgs release supporting `x86_64-darwin`; support and
binaries end with the release branch at the end of 2026.
`allowDeprecatedx86_64Darwin` suppresses the warning. Flakes must pass it
through an explicit Nixpkgs import, not `~/.config/nixpkgs/config.nix`.

```nix
import nixpkgs {
  system = "x86_64-darwin";
  config.allowDeprecatedx86_64Darwin = true;
}
```

### Top-level desktop package scopes

In nixos-26.05, MATE and Xfce packages move to top-level attributes, and `xorg`
is deprecated in favor of top-level packages. Use, for example, `pkgs.caja` and
`pkgs.xfce4-whiskermenu-plugin`. The compatibility `xfce` scope is scheduled
for removal in 26.11.

### Stateless `command-not-found`

In nixos-26.05, when a Nixpkgs source contains `programs.sqlite`, as channel
tarballs do, `command-not-found` enables itself and uses that source database
without mutable state.

### Nixpkgs expression migrations

In nixos-26.05:

- replace `xfce.mkXfceDerivation` with `stdenv.mkDerivation`;
- replace `mpv-unwrapped.scripts` and `.wrapper` with `mpvScripts` and
  `mpv.override`;
- replace deprecated `fetchFromSavannah` with `fetchgit` or a release mirror;
- flatten nested lists in build and runtime inputs.
