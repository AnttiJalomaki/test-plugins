# Stores, Builds, Transports, and Caches

Use this reference for local-store integrity, garbage collection, builder
execution, remote stores, SSH transport, substituters, cache metadata, and S3.

## Store layout, integrity, and garbage collection

### Nix 2.0 is the daemon protocol floor

Since 2.32.0, Nix no longer supports daemon worker-protocol peers older than
Nix 2.0, the first release using protocol version 18. Upgrade clients and
daemons to at least Nix 2.0 before mixing them with newer installations.

### Durable store-path registration

Since 2.25.0, `fsync-store-paths = true` durably writes a new store path before
registering it as valid, reducing corruption after a crash or power loss. It
defaults to `false`.

### Host store paths in chroot stores

Since 2.27.0, evaluation against a chroot store sees the union of host and
chroot `/nix/store` content. Host-store inputs remain accessible, including
through `builtins.path` and `builtins.filterSource`.

### Build directories moved under the Nix state directory

Since 2.30.0, temporary build directories do not follow `$TMPDIR` or default to
`/tmp`. `build-dir` defaults to `builds` beneath `$NIX_STATE_DIR`, normally
`/nix/var/nix/builds`. Update storage provisioning, monitoring, and cleanup
tools.

### Temporary build directory names are opaque

Since 2.32.0, build directory paths omit the derivation name. Monitoring tools
must not infer derivation identity from a temporary directory name.

### Runtime GC roots without `/proc` scanning

Since 2.34.0, `nix store roots-daemon` serves runtime GC roots through a Unix
socket, allowing collection when the primary daemon lacks `CAP_SYS_PTRACE`.
Select it with `use-roots-daemon`. Both that setting and tolerant GC require
the experimental `local-overlay-store` feature.

### Tolerating undeletable local-store paths

Since 2.34.0, experimental local-store setting
`ignore-gc-delete-failure = true` makes GC warn and continue when an
unprivileged process cannot delete a path.

### Store-path lookup and copy in the C API

Since 2.34.0, `nix_store_query_path_from_hash_part()` resolves a hash part to a
full store path, and `nix_store_copy_path()` copies one path between stores with
repair and signature-check controls.

## Build execution and builders

### `libnixstore` no longer knows the Nix binary directory

Since 2.25.0, separately packaged `libnixstore` cannot supply a useful default
path for `build-hook`. Linking applications that use remote builds must put the
Nix executables on `PATH` or set `build-hook`. Perl bindings no longer expose
`getBinDir`.

### Meson/Ninja source builds

Since 2.26.0, Nix itself builds with Meson and Ninja; the Make-based source
build was removed. Migrate source-build and packaging automation.

### Automatic core detection with `build-cores = 0`

Since 2.31.0, `build-cores = 0` detects available CPUs, matching an unset
setting. Builders receive the detected count through `NIX_BUILD_CORES`, not a
literal zero.

### External derivation builders

Since 2.32.0, experimental `external-builders` can delegate selected system
types to helper programs, for example to use QEMU for another platform.

### Unprivileged namespace wrapper

Since 2.34.0 on Linux, `libexec/nix-nswrapper` can run the daemon with full
sandboxing inside an unprivileged user namespace. Allocate build-user UIDs and
GIDs in `/etc/subuid` and `/etc/subgid`; Nixpkgs exposes `nix.daemonUser` and
`nix.daemonGroup`.

## Copying, roots, and signatures

### `nix copy` can create GC roots

Since 2.26.0, `nix copy` accepts `--profile` and `--out-link`. A profile points
to the top-level copied path; an output link creates symlinks to top-level
paths. Creating roots during the copy closes the concurrent-GC window.

```sh
nix copy --from ssh://server \
  --profile /nix/var/nix/profiles/system "$path"
```

### Multiple signing keys in store URIs

Since 2.29.0, the `secret-keys` store URI parameter accepts comma-separated key
files so copied paths can be signed by multiple keys during rotation.

```sh
nix copy --to 'file:///tmp/store?secret-keys=/tmp/key1,/tmp/key2' "$path"
```

### No mixed builds for incomplete substituted closures

Since 2.30.0, Nix does not combine an available substituted object with local
builds for references missing from that substituter. Configure an overlay cache
alongside the underlying cache or clients may build more paths locally.

```ini
substituters = https://overlay.example https://cache.nixos.org
```

## SSH and Git transport

### Shell-style parsing for `NIX_SSHOPTS`

Since 2.26.0, `NIX_SSHOPTS` parses spaces and quotes like a shell, allowing
multiword options such as quoted proxy commands across SSH-based Nix commands.

```sh
export NIX_SSHOPTS='-o ProxyCommand="ssh -W %h:%p ..."'
```

### Git LFS honors SSH transport configuration

Since 2.31.0, Git LFS over SSH honors `NIX_SSHOPTS` and the URL port. It also
uses the `git-lfs-authenticate` response to locate an API endpoint not exposed
on port 443.

### Ports in SSH store URIs

Since 2.31.0, SSH store references in `--store`, related flags, and
remote-builder configuration can include a port with hostnames, IPv4, or
bracketed IPv6:

```text
ssh://user@example.com:2222
ssh-ng://[b573:6a48:e224:840b:6007:6275:f8f7:ebf3]:22
```

### Percent-encoded IPv6 zone identifiers

Since 2.31.0, scoped IPv6 URI literals must encode the `%` zone separator as
`%25`, for example `[fe80::1%2518]`. A literal percent form is invalid.

## Logs and diagnostics

### Mirroring logs as JSON

Since 2.30.0, `json-log-path` copies every Nix log message as JSON to a file or
Unix domain socket.

```ini
json-log-path = /var/log/nix.json
```

## HTTP substituters and fetchers

### TLS verification for the builtin derivation fetcher

Since 2.25.0, `<nix/fetchurl.nix>`—the `builtin:fetchurl` derivation
builder—verifies TLS certificates. Servers with invalid certificates fail.
This is separate from evaluation-time `builtins.fetchurl`, which was not
affected.

### HTTP substituter connection timeout

Since 2.29.0, `connect-timeout` defaults to 5 seconds instead of unlimited.
Override it for intentionally slow or intermittently reachable caches.

### Compressed HTTP cache metadata

Since 2.32.0, HTTP binary caches can compress `.narinfo`, `.ls`, and build-log
uploads with `narinfo-compression`, `ls-compression`, and `log-compression`.
`Content-Encoding` advertises the codec and compatible clients decompress it.

```sh
nix copy --to \
  'http://cache.example?narinfo-compression=gzip&ls-compression=gzip' "$path"
nix store copy-log --to 'http://cache.example?log-compression=br' "$path"
```

### Configurable binary-cache metadata lifetime

Since 2.34.0, `narinfo-cache-meta-ttl` controls how many seconds
`/nix-cache-info` is cached locally; the default is seven days.
`nix store info --refresh` forces a refreshed cache-validity check.

### Client-certificate authentication for HTTPS caches

Since 2.34.0, HTTPS substituter URLs accept `tls-certificate` and
`tls-private-key` for mutual TLS:

```text
https://cache.example?tls-certificate=/path/cert.pem&tls-private-key=/path/key.pem
```

### HTTP content-encoding compatibility

Since 2.34.0, decompression follows linked libcurl capabilities. `deflate` and
deprecated `x-gzip` join `br`, `zstd`, and `gzip`; nonstandard `xz` and `bzip2`
encodings are rejected. Distribution builds must link libcurl with every codec
they intend to support.

## S3 binary caches

### STS credentials for S3 binary caches

Since 2.29.0, the S3 binary-cache client supports the STS profile credential
provider, including credentials established by `aws sso login`.

### S3 multipart upload settings

Since 2.33.0, S3 cache traffic uses HTTP through curl SigV4. Authenticated
operation requires curl 7.75.0 or newer and `aws-crt-cpp`; without the latter,
only public buckets work. Multipart uploads use:

- `multipart-upload`, default `false`;
- `multipart-threshold`, default 100 MiB;
- `multipart-chunk-size`, default and minimum 5 MiB.

`buffer-size` is now an alias for `multipart-chunk-size`.

### Version-pinned S3 objects

Since 2.33.0, S3 URLs accept `versionId` to pin an object in a versioned bucket:

```text
s3://bucket/key?region=us-east-1&versionId=abc123def456
```

### S3 upload storage classes

Since 2.33.0, `storage-class` applies to normal and multipart cache uploads. If
omitted, the bucket default applies.

```sh
nix copy --to 's3://my-bucket?storage-class=INTELLIGENT_TIERING' "$path"
```

### S3 addressing style

Since 2.34.0, `addressing-style=auto` uses virtual-hosted URLs for standard AWS
endpoints and path style for custom endpoints or dotted bucket names. `path`
forces the deprecated path form. `virtual` forces virtual-hosted addressing
and cannot be used with dotted bucket names.

### Container-native S3 credentials

Release 2.34.2 restores the STS WebIdentity provider used by EKS IRSA, OIDC
workflows, and `AssumeRoleWithWebIdentity`, plus ECS metadata credentials used
by ECS tasks and EKS Pod Identity. Web identity uses
`AWS_WEB_IDENTITY_TOKEN_FILE` and role variables. Container metadata uses
`AWS_CONTAINER_CREDENTIALS_RELATIVE_URI` or
`AWS_CONTAINER_CREDENTIALS_FULL_URI`.
