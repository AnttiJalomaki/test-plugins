---
name: nix-knowledge-patch
description: Nix
version: null
license: MIT
metadata:
  author: Nevaberry
---

# Nix Compatibility Guide

Load this skill when changing Nix expressions, Nix CLI automation, NixOS
modules, Nixpkgs packages, binary-cache infrastructure, or Home Manager
configuration. Use it during upgrades, when reviewing code written against an
older interface, and when a current command behaves differently than expected.

The reference files are the source of detail. Start with the quick checks below,
then open the reference matching the work at hand. Treat project expressions,
locked inputs, state-version declarations, and observed evaluation results as
authoritative for the specific deployment.

## Reference index

| Reference | Topics |
| --- | --- |
| [Nix language, CLI, flakes, and APIs](references/nix-language-cli-flakes.md) | Evaluation semantics, CLI output, flake inputs and locks, REPL, installer, C and C++ APIs |
| [Stores, builds, transports, and caches](references/stores-builds-caches.md) | Store durability and GC, builders, SSH, HTTP and S3 substituters, logs, credentials |
| [NixOS systems and services](references/nixos-systems-services.md) | Rebuilds, boot and initrd, networking, service modules, security, databases, system migrations |
| [Nixpkgs packaging](references/nixpkgs-packaging.md) | Builders, hooks, library APIs, language ecosystems, package scopes, platform changes |
| [Home Manager](references/home-manager.md) | Activation, profiles, state-gated defaults, module renames, XDG and Darwin behavior |

## Upgrade triage

Before changing code:

1. Identify the Nix, Nixpkgs, NixOS, and Home Manager revisions independently.
   They evolve on different schedules.
2. Read `system.stateVersion` and `home.stateVersion`; do not bump either merely
   to silence an option warning.
3. Inspect `flake.lock`, explicit package pins, cache URLs, and remote-builder
   URIs before assuming defaults.
4. Search the references for every removed option, renamed command, or changed
   JSON field reported by evaluation or CI.
5. Evaluate first, build second, and activate last. For system changes, keep a
   bootable generation and a rollback path.

## Breaking evaluation and CLI checks

- Signed integer overflow is an evaluation error. JSON integers must fit in a
  signed 64-bit value, and negative numeric `nixConfig` values are rejected.
- Relative path flake locks are not readable by older Nix clients. Coordinate
  lock-file format changes across all machines that consume the repository.
- Lock generation does not consult user or system flake registries. Pin input
  URLs or use an explicit command-line override for reproducible resolution.
- Short, URL, and absolute path literal checks are tri-state lints. Prefer the
  stable lint settings and decide explicitly between `ignore`, `warn`, and
  `fatal`.
- A zero-argument `nix fmt` call is distinct from `nix fmt .`; the configured
  formatter decides what no arguments mean.
- JSON intended for machines needs an explicit schema where the command offers
  one. Do not parse terminal pretty-printing or human-readable size units.
- Derivation JSON writers must emit the current envelope and structured input
  and content-address fields; older input formats are not accepted.
- `nix flake check` may validate substitutable outputs without downloading them.
  Success does not prove every result exists in the local store.
- Relative paths in `file:` tarball references are invalid. Use an absolute
  path or a different reference form.

## Store and build safety

- Build directories live under the Nix state directory and have opaque names.
  Do not discover active derivations by scanning `/tmp` or parsing directory
  names.
- `build-cores = 0` means automatic CPU detection; builders receive the detected
  count through `NIX_BUILD_CORES`.
- Incomplete closures from a substituter are not combined with local builds.
  Configure overlay caches together with the cache that provides their
  references.
- Use `nix copy --profile` or `--out-link` when copied paths need immediate GC
  protection.
- Turn on durable store-path registration only after considering its I/O cost;
  it is a crash-consistency setting, not a substitute for backups.
- External derivation builders, tolerant GC, and the runtime roots daemon are
  experimental facilities with explicit feature and privilege requirements.
- If an application links `libnixstore` and performs remote builds, ensure Nix
  executables are on `PATH` or set `build-hook` explicitly.

## Cache and transport checks

- HTTP substituters have a finite default connection timeout. Override it for
  deliberately slow endpoints.
- Compression for cache metadata and logs is conveyed by `Content-Encoding`;
  use codecs supported by the linked libcurl.
- HTTPS substituters can use a client certificate and key. Keep both paths out
  of world-readable configuration.
- S3 authentication and upload behavior depend on the Nix build's curl and AWS
  CRT support. Confirm OIDC, container metadata, multipart settings, addressing
  style, and storage class in the deployment environment.
- SSH store URIs accept explicit ports. Bracket IPv6 literals and percent-encode
  a scoped address's zone separator.
- `NIX_SSHOPTS` is shell-parsed. Quote multiword options as one shell argument
  and test them through both ordinary Git and Git LFS paths.

## NixOS migration checks

- The systemd initrd is the default. Verify encrypted-root device names, stable
  root paths, and any LVM-on-LUKS timeout before rebooting.
- `nixos-rebuild-ng` is the normal rebuild implementation. Remove obsolete
  opt-in and switch implementation toggles.
- Services that require connectivity must explicitly want and order themselves
  after `network-online.target`.
- Core module types are stricter: declare filesystem types, use lists where
  required, and expect unknown video drivers to fail evaluation.
- RFC 42-style `settings` attributes replace many free-form `extraConfig`
  strings. Migrate values structurally instead of copying raw fragments.
- Changing the D-Bus implementation is a switch inhibitor and needs a reboot.
- Secret-bearing module options increasingly require files or credential
  injection. Never put private keys or application secrets into the Nix store.
- Database and application defaults are state-gated. Major-version upgrades for
  Nextcloud and extension changes for Immich require deliberate stepping.
- Removed kernels, filesystems, legacy network tools, and module paths require a
  real replacement; an option rename alone may not preserve behavior.

## Nixpkgs packaging checks

- Modern Python builders require an explicit build format and build system.
- Rust packages use `cargoHash` and stable vendoring helpers; regenerate hashes
  after Cargo-format changes.
- Go packages use `buildGoModule`; place `CGO_ENABLED` under `env` and do not
  pass `GOOS` or `GOARCH` as builder arguments.
- `buildEnv` uses fixed-point arguments and structured attributes. Put
  derivation-specific arguments under `derivationArgs`.
- `stdenv.mkDerivation` requires `env` to be an attribute set. A variable
  literally named `env` belongs at `env.env`.
- Broken-symlink checks reject dangling, reflexive, and build-directory links.
  Disable the check only for an intentional output contract.
- Use current Nixpkgs library, command-line rendering, fetcher, and package-scope
  APIs. Removed aliases often differ semantically from their replacements.
- Default compiler and runtime changes can break unpinned packages. Pin only
  when necessary and preserve the reason in the expression.
- Nested build-input lists are deprecated; flatten them.

## Home Manager migration checks

- Generated activation scripts should not own profile updates. Callers that run
  them directly must manage the profile.
- User-service activation now restarts services by default; the legacy mode is
  invalid.
- Minimal module mode imports only Home Manager basics. Import every program or
  service module used by the configuration.
- State-gated XDG, application-copying, plugin-language, and window-manager
  defaults can move files or reinterpret configuration.
- Prefer structured SSH settings and the common SSH-auth-socket integration.
  Migrate profile-scoped Firefox and Anki configuration to the profile itself.
- Login-time activation is available for homes unavailable during boot.
- Automatic upgrades no longer imply an input update. Add a pre-switch update
  command only when that policy is desired.

## Verification patterns

For expressions and flakes:

```sh
nix flake check
nix eval --show-trace .#nixosConfigurations.host.config.system.build.toplevel.drvPath
```

For a NixOS host:

```sh
nixos-rebuild build
nixos-rebuild test
```

For a package:

```sh
nix build --print-build-logs .#package
```

For output consumed by automation, redirect stdout and select explicit JSON
format and pretty-print flags. Validate the result against the command's chosen
schema rather than relying on presentation defaults.

## Decision rules

- When a state version gates a default, preserve the current state version and
  configure the desired behavior explicitly unless the user is intentionally
  adopting all migrations attached to the newer state.
- When an option is removed, follow the replacement described in the relevant
  reference; do not recreate removed free-form configuration through string
  concatenation.
- When cache behavior differs between hosts, compare Nix build features,
  credential-provider environment, URI parameters, and proxy or SSH options.
- When a flake update changes more than the named input, inspect nested lock
  provenance before forcing a second update.
- When activation can interrupt access, use a test activation or a recoverable
  console and verify inhibitors rather than bypassing them by default.
