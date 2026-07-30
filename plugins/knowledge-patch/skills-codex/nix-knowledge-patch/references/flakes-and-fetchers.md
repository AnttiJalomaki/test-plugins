# Flakes and Fetchers

## Input declarations

### Relative paths

A flake can refer to another flake in the same repository with a relative
`path:` input (since 2.26.0):

```nix
inputs.foo.url = "path:./foo";
```

This changes the lock-file representation. Older Nix versions cannot consume
a lock containing relative-path input locks, so account for the oldest client
that must read the repository.

### Git submodules and LFS

A Git-backed flake can declare its own submodule requirement (since 2.27.0):

```nix
inputs.self.submodules = true;
```

Callers no longer need to supply `submodules = true`.

The Git fetcher can materialize Git LFS objects when `lfs` is enabled (since
2.27.0). A flake may declare `inputs.self.lfs = true`, or a Git URL may add
`lfs=1`:

```sh
nix flake prefetch 'git+ssh://git@example.com/repo.git?lfs=1'
```

Git LFS over SSH honors `NIX_SSHOPTS` and the URL's port (since 2.31.0). It
also follows the `git-lfs-authenticate` response to an API endpoint that need
not be exposed on port 443.

Experimental Git-hashed store objects can use SHA-256 as well as SHA-1 (since
2.31.0).

### Non-flake source subdirectories

Inputs with `flake = false` expose their parent source's `sourceInfo` (since
2.30.0), distinguishing the containing source from an input below it. Select a
subdirectory with `?dir=subdir`:

```nix
inputs.data = {
  url = "path:./vendor?dir=subdir";
  flake = false;
};
```

## Lock-file resolution

When generating a lock for an indirect reference such as `nixpkgs`, Nix
ignores system and user flake registries (since 2.26.0). It uses the global
registry and command-line `--override-flake` values. Pin the URL when
resolution must be reproducible:

```nix
inputs.nixpkgs.url = "github:NixOS/nixpkgs/nixos-24.11";
```

When an input reference changes during an update, Nix consults that input's
own lock file for its nested inputs (since 2.31.0). It preserves the versions
chosen there rather than fetching each nested input's latest version.

Use `nix registry resolve` to resolve an indirect registry name and print the
resulting flake reference without fetching or evaluating it (since 2.33.0):

```console
$ nix registry resolve nixpkgs
github:NixOS/nixpkgs/nixpkgs-unstable
```

## Prefetch, archive, show, and check

`nix flake prefetch` accepts `--out-link` (since 2.27.0), so the prefetch can
create a root for its result:

```sh
nix flake prefetch --out-link ./result <flake-reference>
```

`nix flake prefetch-inputs` fetches all inputs in parallel (since 2.31.0). It
avoids serialized on-demand fetches but may fetch inputs that evaluation would
not use.

`nix flake show` skips outputs requiring import-from-derivation and continues
showing the rest of the flake (since 2.29.0), rather than failing the entire
command.

`nix flake archive` accepts `--no-check-sigs` (since 2.30.0). Use it when
copying an archive directly to a remote store whose signature checks would
otherwise block the operation:

```sh
nix flake archive --to ssh-ng://builder --no-check-sigs .
```

This deliberately relaxes verification; scope it to a trusted transfer path.

When a derivation is substitutable, `nix flake check` may skip downloading its
output (since 2.32.0). A successful check therefore does not guarantee that
every checked output exists locally.

`nix flake clone` supports arbitrary input types rather than only Git-oriented
ones (since 2.33.0), including tarball-backed flakes such as those hosted on
FlakeHub.

## Tarballs and channels

The channel server implements the Lockable HTTP Tarball Protocol (since
nixos-25.05), allowing a channel's `nixexprs.tar.xz` to be a flake input:

```nix
inputs.nixpkgs.url =
  "https://channels.nixos.org/nixos-25.05/nixexprs.tar.xz";
```

Built-in channel URLs use `https://channels.nixos.org/` (since 2.33.0). The old
`https://nixos.org/channels/` location redirects for now, but migrate stored
URLs and network allowlists before that redirect disappears.

Tarball references using `file:` must use absolute paths (since 2.34.0).
Relative `file:` paths are rejected rather than accepted ambiguously.

S3 source URLs accept `versionId` (since 2.33.0), allowing a versioned bucket
object to be pinned reproducibly:

```text
s3://bucket/key?region=us-east-1&versionId=abc123def456
```

## Fetcher transport and policy

The derivation builder in `<nix/fetchurl.nix>`, also known as
`builtin:fetchurl`, verifies TLS certificates (since 2.25.0). HTTPS requests to
servers with invalid certificates fail. Evaluation-time `builtins.fetchurl`
was not part of the affected old behavior.

Nixpkgs can configure fetchers globally (since nixos-25.11):

- `rewriteURL` and `hashedMirrors` apply to `fetchurl`.
- `gitConfig` or `gitConfigFile` apply to every `fetchgit`.
- `npmRegistryOverrides` or `npmRegistryOverridesString` apply to
  `fetchNpmDeps`.
- Individual `fetchgit` calls accept `gitConfigFile` and `rootDir`.
- Individual `fetchNpmDeps` calls accept `npmRegistryOverridesString`.

Keep policy inputs explicit in evaluations that require reproducible source
selection.

## Chroot stores

With a chroot store, the evaluator sees the union of host and chroot
`/nix/store` contents (since 2.27.0). Host-store inputs remain accessible, and
`builtins.path` plus `builtins.filterSource` operate correctly in that store
mode.
