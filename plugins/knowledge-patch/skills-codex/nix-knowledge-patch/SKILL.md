---
name: nix-knowledge-patch
description: Nix
version: null
license: MIT
metadata:
  author: Nevaberry
---


# Nix Knowledge Patch

Use this skill when changing Nix, NixOS, Nixpkgs, or Home Manager code whose
behavior may depend on recent CLI, evaluator, store, module, or packaging
changes. Establish which product and release the project actually uses before
applying version-sensitive guidance.

## Working method

1. Inspect `flake.nix`, `flake.lock`, channel pins, `system.stateVersion`,
   `home.stateVersion`, package expressions, and deployment configuration.
2. Identify whether the task concerns Nix itself, NixOS modules, Nixpkgs
   packaging, or Home Manager; these products evolve on different schedules.
3. Read the matching topic reference before editing. Follow repository code,
   evaluated options, and tests when they disagree with compatibility notes.
4. Treat state versions as compatibility switches, not as declarations of the
   currently installed package set.
5. Check generated JSON, lock files, store paths, and command output with
   machine-readable interfaces rather than parsing presentation-oriented text.
6. Validate migrations by evaluating the configuration and building the
   narrowest affected derivation or system closure.

## Reference index

| Reference | Topics |
| --- | --- |
| [core-cli-evaluation.md](references/core-cli-evaluation.md) | Evaluator semantics, CLI output, configuration, REPL, installer, profiling |
| [flakes-and-fetchers.md](references/flakes-and-fetchers.md) | Flake inputs and locks, Git features, registries, tarballs, prefetching |
| [stores-caches-and-remote-builds.md](references/stores-caches-and-remote-builds.md) | GC roots, substituters, S3, SSH stores, logging, store durability |
| [native-api-and-source-builds.md](references/native-api-and-source-builds.md) | C and C++ APIs, installed headers, source build system |
| [nixos-system-and-services.md](references/nixos-system-and-services.md) | Boot, networking, security, service-module and rebuild migrations |
| [nixpkgs-packaging.md](references/nixpkgs-packaging.md) | Builders, hooks, language ecosystems, library and package-set migrations |
| [home-manager.md](references/home-manager.md) | Activation, state-gated defaults, module and configuration migrations |

## High-priority compatibility checks

### Nix expressions and command output

- Signed 64-bit integer overflow is an evaluation error; JSON integers above
  that range are rejected.
- Use `__structuredAttrs = true` on `builtins.derivation`; do not construct
  structured derivations by putting serialized JSON in `__json`.
- A zero-argument `nix fmt` invocation is distinct from `nix fmt .`.
- `--json` formatting may depend on whether stdout is a terminal. Use
  `--pretty` or `--no-pretty` when byte layout matters.
- Pass `--json-format` to `nix path-info --json`. Its version 2 schema uses
  store-path basenames and structured content addresses.
- `nix derivation show` emits schema version 4; its inputs are nested below
  `inputs`, and `nix derivation add` rejects older schemas.
- Do not parse fixed size units or infer a derivation name from a temporary
  build-directory name.

### Flakes and locks

- Relative `path:` flake inputs produce locks that older Nix versions cannot
  consume.
- Lock generation for indirect inputs ignores local system and user
  registries; pin explicit URLs for reproducibility.
- A flake can declare `inputs.self.submodules = true` or `.lfs = true`.
- Updating an input preserves nested versions from that input's own lock file.
- `nix flake check` may validate a substitutable derivation without realizing
  its output locally.
- Relative paths in `file:` tarball references are rejected.

### Store and cache operation

- Set `fsync-store-paths = true` when new store paths must be durable before
  registration.
- Create a root as part of `nix copy` with `--profile` or `--out-link` when a
  concurrent garbage collection could race the copy.
- Configure overlay substituters together with their underlying cache; Nix no
  longer mixes an incomplete substituted closure with local builds.
- HTTP substituter connection attempts time out after five seconds by default.
- Configure URI compression and S3 upload parameters deliberately; accepted
  content encodings and multipart behavior have changed.
- Nix 2.32 cannot speak to daemon peers older than Nix 2.0.

### NixOS state and activation

- `system.stateVersion` must have the `YY.MM` form. Change it only as part of a
  deliberate compatibility migration.
- Services requiring connectivity must explicitly want and follow
  `network-online.target`.
- The systemd initrd is the default stage 1 implementation. Use stable
  root-device paths and valid mapped LUKS device names.
- `dbus-broker` is the default D-Bus implementation; switching implementation
  requires a reboot.
- `nixos-rebuild-ng` is the default. Remove obsolete Rust-switch settings and
  use the temporary opt-out only when necessary.
- A dependency on `postgresql.target` guarantees writable readiness and
  completed initialization; `postgresql.service` guarantees only read access.
- Initial self-signed ACME certificates are always created. Depend on the
  certificate service for syntactic validity and use the renewal-order service
  for ordering.
- Switch inhibitors can reject incompatible generation changes. Use
  `NIXOS_NO_CHECK=1` only as an intentional force override.

### NixOS module breakages

- Replace removed free-form systemd, coredump, sleep, logind, and service
  configuration strings with their structured `settings` forms.
- OpenSSH `AcceptEnv` is a list, every filesystem needs an explicit `fsType`,
  and unknown X server video drivers fail evaluation.
- Configure server and client Postfix TLS chains in `services.postfix.settings`;
  the old certificate and key options are gone.
- Explicitly configure NetworkManager VPN plugins and imperative-container
  boot support.
- Migrate renamed modules, including River, DNSCrypt, Bluesky PDS,
  separated-debuginfod, and LXD-to-Incus.
- Treat secret inputs as files or runtime-expanded values; do not place private
  Grafana, OAuth2 Proxy, or Yggdrasil values in the store.
- Step Nextcloud through every supported major release before selecting the
  state-gated default.

### Nixpkgs package expressions

- Use `cargoHash` and `rustPlatform.fetchCargoVendor`; `cargoSha256` and the
  Cargo-format-dependent tarball fetcher are obsolete.
- Use `buildGoModule`; place `CGO_ENABLED` under `env` and do not pass `GOOS`
  or `GOARCH` as builder arguments.
- Modern Python packages must explicitly opt into `pyproject` and list their
  build system.
- Add `postgresql.pg_config` or `libpq.pg_config` to native build inputs when
  `pg_config` is required.
- `stdenv.mkDerivation` requires `env` to be an attribute set. A variable
  literally named `env` belongs at `env.env`.
- Use structured attributes for `buildEnv`; place extra derivation arguments
  under `derivationArgs`.
- Broken-symlink checks reject dangling, reflexive, and build-directory links.
  Disable the check only for intentional output structure.
- Flatten nested dependency lists and migrate removed Nixpkgs library helpers
  to their named replacements.

### Home Manager

- User services start by default; the removed `"legacy"` activation mode is an
  evaluation error.
- Direct callers of generated activation scripts must manage the profile
  update themselves.
- `home-manager switch --rollback` and `--specialisation NAME` preserve the
  profile while selecting the target generation.
- Minimal mode imports only Home Manager's essential modules; import every
  additional module explicitly.
- State-gated defaults affect signing, password-store and application paths,
  XDG layout, Neovim and Hyprland configuration formats, and automation.
- Prefer `programs.ssh.settings`; migrate per-profile Firefox and Anki
  configuration and renamed service modules.

## Focused validation

### Evaluate before switching

```sh
nix flake check
nix eval .#nixosConfigurations.host.config.system.build.toplevel.drvPath
nix build .#nixosConfigurations.host.config.system.build.toplevel
```

Remember that a successful flake check may leave substitutable outputs absent
from the local store. Build the output explicitly when later checks require it.

### Stabilize machine-readable output

```sh
nix eval --json --no-pretty --expr '{ answer = 42; }'
nix path-info --json --json-format 2 /nix/store/...
nix derivation show /nix/store/...drv
```

Keep schema-version handling explicit and use each response's `storeDir`
instead of assuming `/nix/store`.

### Inspect option migrations

```sh
nixos-option --show-trace services.openssh.settings
nix eval .#nixosConfigurations.host.config.system.stateVersion
nix eval .#homeConfigurations.user.config.home.stateVersion
```

For flake configurations, use the flake-aware option lookup and evaluate the
specific option subtree affected by the migration.

### Test store and remote behavior

```sh
nix copy --from ssh://builder --out-link ./result /nix/store/...
nix store info --refresh
nix flake prefetch-inputs .
```

Confirm that remote store ports, SSH options, credentials, substituter order,
and cache compression match the selected transport.

## Guardrails

- Do not advance state versions merely to silence an option warning.
- Do not put credentials, private keys, or expanded secrets into Nix
  expressions that become world-readable store paths.
- Do not assume a compatibility alias will remain; migrate to the canonical
  command, option, package, or helper.
- Do not parse terminal-oriented output when a versioned JSON interface exists.
- Do not rely on paths, defaults, or service ordering that the configuration
  can state explicitly.
