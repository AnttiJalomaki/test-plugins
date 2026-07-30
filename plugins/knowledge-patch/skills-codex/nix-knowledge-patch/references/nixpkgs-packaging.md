# Nixpkgs Packaging

## General derivation construction

### Environment attributes

`stdenv.mkDerivation` and related builders require `env` to be an attribute set
as of nixos-25.11. To create an environment variable literally named `env`,
nest it:

```nix
stdenv.mkDerivation {
  env.env = "value";
}
```

`requireFile` treats `message` and `url` as literal strings in nixos-26.05,
rather than feeding them through a Bash here-document. Forms such as `$PWD` no
longer expand and need no shell-specific escaping.

Nested lists in build and runtime inputs are deprecated in nixos-26.05; flatten
them.

### Substitution helpers

`substituteAll` and `substituteAllFiles` are deprecated in nixos-25.05. Use
`replaceVars`; the older helpers are scheduled for removal.

### `buildEnv`

`buildEnv` takes fixed-point `finalAttrs: { ... }` arguments in nixos-25.11.
Its custom result `.override` is deprecated. Put extra `stdenv.mkDerivation`
arguments under `derivationArgs`; direct `nativeBuildInputs` and `buildInputs`
survive only through a compatibility layer.

Nixos-26.05 completes the migration: `buildEnv` packages use structured
attributes only, with `__structuredAttrs = true`.

### Package main programs

`meta.mainProgram` determines `NIX_MAIN_PROGRAM` in nixos-25.11, so changing it
can rebuild a package. `versionCheckHook` can fail when `pname` differs from the
selected main program instead of silently checking the `pname` executable.

## Output validation and hardening

The `no-broken-symlinks` hook added in nixos-25.05 rejects dangling and
reflexive output links. Set `dontCheckForBrokenSymlinks = true;` only for a
derivation that intentionally needs them.

In nixos-25.11 the same hook also rejects links into `$TMPDIR`, normally
`/build`.

The `pie` hardening flag is removed in nixos-25.11 because toolchains enable
PIE by default. A package that truly cannot use PIE must add `-no-pie` through
`CFLAGS`.

A derivation combining `separateDebugInfo` with `allowedReferences`,
`allowedRequisites`, `disallowedReferences`, or `disallowedRequisites` must set
`__structuredAttrs = true` as of nixos-25.11. Reference allow/deny checks do
not apply to the generated `debug` output.

Glibc 2.42 no longer makes the stack executable merely because a loaded shared
library requests it (nixos-26.05). Prefer rebuilding with:

```nix
env.NIX_LDFLAGS = "-z,noexecstack";
```

Alternatively clear an erroneous flag with `patchelf --clear-execstack`. Use
`GLIBC_TUNABLES=glibc.rtld.execstack=2` only per process for code that truly
requires an executable stack.

## C, C++, graphics, and platform packages

Applications linked against different Mesa versions can coexist starting in
nixos-25.05. A package that needs GBM or DRI metadata should depend on
`libgbm` or `dri-pkgconfig-stub`, respectively, rather than Mesa itself.

Nixpkgs 25.11 requires Nix 2.18 or newer to evaluate. Linux kernel packages
move every in-tree kernel module to a separate `modules` output; consumers
must not assume the modules are in the primary output.

Nixpkgs 25.11 requires macOS 14.0 or newer and defaults to SDK 14.4. Darwin
builds use the system libc++; raise a package's deployment target when it
needs newer C++ library features.

Nixos-26.05 is the final Nixpkgs release supporting `x86_64-darwin`; release
branch support and binaries end at the end of 2026. Set
`allowDeprecatedx86_64Darwin` to suppress the warning. Flake users must pass it
to an explicit Nixpkgs import rather than relying on
`~/.config/nixpkgs/config.nix`:

```nix
import nixpkgs {
  system = "x86_64-darwin";
  config.allowDeprecatedx86_64Darwin = true;
}
```

The default compiler moves from GCC 14 to GCC 15 in nixos-26.05. This can
surface upstream incompatibilities in expressions that did not pin a compiler.

## Rust

Cargo 1.84 invalidated existing `cargoHash` values in nixos-25.05.
`rustPlatform.fetchCargoVendor` replaces the Cargo-format-dependent
`fetchCargoTarball`. `rustPlatform.buildRustPackage` no longer accepts
`cargoSha256`; regenerate and use `cargoHash` for out-of-tree packages.

## Go

`buildGoPackage` is removed in nixos-25.05; use `buildGoModule`. Put
`CGO_ENABLED` at `env.CGO_ENABLED`. Direct `GOOS` and `GOARCH` builder
arguments now error. Use the new `goSum` and self-referencing `finalAttrs`
inputs when needed.

```nix
env.CGO_ENABLED = "1";
```

## Python

Python build hooks represent flags like `stdenv.mkDerivation` in nixos-25.05:
space-separated variables when structured attributes are disabled and Bash
arrays when they are enabled, without Bash-evaluating the values.
`pytestFlags` and `unittestFlags` replace the compatibility-only
`pytestFlagsArray` and `unittestFlagsArray`.

`buildPythonPackage` and `buildPythonApplication` require an explicit format in
nixos-25.11. For a modern setuptools package, set `pyproject = true` and
provide `build-system = [ setuptools ]`. Passing `stdenv` directly is
deprecated; override the helper:

```nix
(buildPythonPackage.override { stdenv = customStdenv; }) {
  pyproject = true;
  build-system = [ setuptools ];
}
```

## PostgreSQL packages

`postgresql` and `libpq` no longer contain `pg_config` in nixos-25.05. Add
`postgresql.pg_config` or `libpq.pg_config` to `nativeBuildInputs`.

PL/Python, PL/Perl, and PL/Tcl are selected through
`postgresql.withPackages`, not the old support overrides:

```nix
nativeBuildInputs = [ postgresql.pg_config ];
postgresql.withPackages (ps: [ ps.plpython3 ])
```

## JavaScript package ecosystems

The default Node.js moves from 22 LTS to 24 LTS in nixos-26.05, potentially
exposing upstream compatibility changes in unpinned packages.

The `nodePackages` set and `node2nix` tooling are removed in nixos-26.05. Use
corresponding top-level packages or current JavaScript packaging helpers.
`yarn2nix`, `mkYarnPackage`, and related tooling are also removed; Yarn 1
packages should use `yarnBuildHook`, `yarnConfigHook`, and `yarnInstallHook`.

`nodejs_latest` names Node 26. `nodejs` is a non-overridable wrapper around
`nodejs-slim` plus npm and Corepack outputs. Refer to or override
`nodejs-slim` attributes directly, and include `nodejs-slim.corepack`
explicitly when using the slim package.

Top-level `fetchPnpmDeps` and `pnpmConfigHook` replace `pnpm.fetchDeps` and
`pnpm.configHook`. Pnpm fetcher versions 1 and 2 are deprecated; regenerate
hashes with version 3. For npm workspaces, set `npmDepsFetcherVersion = 2` on
`buildNpmPackage` to enable packument caching.

## Ruby and other runtime defaults

The default Ruby moves from 3.3 to 3.4 in nixos-26.05. Like the compiler and
Node changes, this can reveal upstream incompatibilities when an expression
does not pin the runtime.

## Fonts and desktop package scopes

The monolithic `nerdfonts` package is split into per-font packages below
`nerd-fonts` in nixos-25.05. Font files also gain a per-font directory below
`share/fonts/{opentype,truetype}/NerdFonts/`; migrate both package attributes
and hard-coded paths.

MATE and Xfce packages move to top-level attributes in nixos-26.05, and the
`xorg` package set is deprecated in favor of top-level packages. Use, for
example, `pkgs.caja` and `pkgs.xfce4-whiskermenu-plugin`. The compatibility
`xfce` scope is scheduled for removal in 26.11.

`xfce.mkXfceDerivation` is deprecated in favor of `stdenv.mkDerivation`.

## Package scopes and Nixpkgs configuration

`lib.packagesFromDirectoryRecursive` rejects unknown arguments and can build
nested scopes mirroring its directory tree as of nixos-25.05.

Nixpkgs configuration functions receive `lib` directly as well as `pkgs` in
nixos-26.05. This avoids depending on the package fixed point and prevents
recursion when only library values are needed:

```nix
{ lib, ... }: {
  allowlistedLicenses = [ lib.licenses.nasa13 ];
}
```

## Library API migrations

Nixos-25.11 removes or replaces these library names:

| Old | Replacement |
| --- | --- |
| `cartesianProductOfSets` | `lib.attrsets.cartesianProduct` |
| `zipWithNames` | `zipAttrsWithNames` |
| `zip` | `zipAttrsWith` |
| `literalExample` | `literalExpression` or `literalMD` |
| `mapAttrsFlatten` | `lib.attrsets.mapAttrsToList` |
| `lib.modules.defaultPriority` | `defaultOverridePriority` |
| `mkPackageOptionMD` | `mkPackageOption` |
| `replaceChars` | `replaceStrings` |
| `lib.sources.path*` | corresponding `lib.filesystem` helpers |
| `lib.types.string` | an appropriate type such as `lib.types.str` |

`lib.strings.isCoercibleToString` splits into `isStringLike` and the broader
`isConvertibleWithToString`.

`types.either` no longer accepts mismatched values when used as a
`freeformType`; this also affects `oneOf`, `number`, and `numbers.*`. Module
authors often need `attrsOf (types.either ...)` so the free-form portion has
the intended shape.

`lib.cli.toGNUCommandLine` and `toGNUCommandLineShell` are deprecated. Use
`lib.cli.toCommandLine`, `toCommandLineShell`, `toCommandLineGNU`, or
`toCommandLineShellGNU`; select a GNU-specific form only for GNU rendering.

## Format helpers and command delivery

Direct use of `pkgs.formats.systemd` is deprecated in nixos-25.11. Instantiate
it like other format helpers:

```nix
systemdFormat = pkgs.formats.systemd { };
```

When `programs.sqlite` is present in the Nixpkgs source, as it is in channel
tarballs, `command-not-found` is enabled automatically and uses the source
database without mutable state (nixos-26.05).

## Neovim and application wrappers

Nixpkgs disables Neovim's Python 3 and Ruby providers by default in
nixos-26.05. Lua dependencies are recorded in generated `init.lua` instead of
`LUA_PATH` wrapper arguments. Commands requiring them must run after
initialization with `-c`; `wrapRc = false` users must load the generated init
file themselves.

`mpv-unwrapped.scripts` and `.wrapper` are replaced by `mpvScripts` and
`mpv.override` in nixos-26.05. `fetchFromSavannah` is deprecated in favor of
`fetchgit` or a release mirror.

## Official formatter name

Use `pkgs.nixfmt` as the stable official formatter package in nixos-25.11.
`pkgs.nixfmt-rfc-style` is deprecated; the older implementation remains
temporarily available as `pkgs.nixfmt-classic`.
