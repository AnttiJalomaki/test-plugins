# Stores, Caches, and Remote Builds

## Store-path lifetime and durability

Enable `fsync-store-paths` to durably write new store paths before registering
them as valid (since 2.25.0). It reduces corruption risk after a crash or power
loss and defaults to `false`.

```ini
fsync-store-paths = true
```

`nix copy` accepts `--profile` and `--out-link` (since 2.26.0).
`--profile` points a profile at the top-level copied path; `--out-link`
creates symlinks to top-level copied paths. Creating the root during the copy
closes the garbage-collection race that exists when rooting afterward.

```sh
nix copy --from ssh://server \
  --profile /nix/var/nix/profiles/system "$path"
```

When a substituter has a store object but lacks something in its closure, Nix
does not combine that partial substitution with local builds for the missing
pieces (since 2.30.0). Configure overlay caches alongside their underlying
cache, or the client may build more paths locally:

```ini
substituters = https://overlay.example https://cache.nixos.org
```

## Garbage collection

`nix store roots-daemon` serves runtime GC roots over a Unix socket (since
2.34.0). It allows collection when the main daemon cannot scan `/proc` because
it lacks `CAP_SYS_PTRACE`. Select it with `use-roots-daemon`; this and the
tolerant-GC setting require the experimental `local-overlay-store` feature.

The local-store setting `ignore-gc-delete-failure` makes GC warn and continue
when an unprivileged process cannot delete a store path (since 2.34.0):

```ini
ignore-gc-delete-failure = true
```

Use it only when leaving undeletable paths is preferable to failing the
collection.

## Remote builders and SSH stores

Separately packaged `libnixstore` cannot infer a Nix binary directory for the
default `build-hook` (since 2.25.0). Programs linking it and using remote
builds must put Nix binaries on `PATH` or set `build-hook` explicitly. The Perl
bindings no longer expose `getBinDir`.

`NIX_SSHOPTS` uses shell-style handling of spaces and quotes (since 2.26.0),
supporting complex options such as quoted proxy commands consistently across
SSH-based Nix commands:

```sh
export NIX_SSHOPTS='-o ProxyCommand="ssh -W %h:%p ..."'
```

SSH store URIs used by `--store`, related flags, and remote-builder
configuration accept ports (since 2.31.0):

```text
ssh://user@example.com:2222
ssh-ng://[b573:6a48:e224:840b:6007:6275:f8f7:ebf3]:22
```

Scoped IPv6 addresses must percent-encode the zone separator as `%25` (since
2.31.0); literal `%` is no longer valid:

```text
[fe80::1%2518]
```

## HTTP substituters

HTTP substituter `connect-timeout` defaults to five seconds instead of having
no limit (since 2.29.0). Override it for intentionally slow or intermittently
reachable caches.

`narinfo-cache-meta-ttl` controls how long `/nix-cache-info` metadata stays
cached locally (since 2.34.0); the default is seven days. `nix store info
--refresh` forces a renewed cache-validity check.

```ini
narinfo-cache-meta-ttl = 3600
```

HTTPS substituter URLs accept `tls-certificate` and `tls-private-key` for
mutual TLS (since 2.34.0):

```text
https://cache.example?tls-certificate=/path/cert.pem&tls-private-key=/path/key.pem
```

HTTP cache metadata and logs can use transparent compression (since 2.32.0):

- `narinfo-compression` controls `.narinfo`.
- `ls-compression` controls `.ls`.
- `log-compression` controls uploaded build logs.

Compression is advertised through `Content-Encoding`, and compatible clients
decompress it:

```sh
nix copy --to \
  'http://cache.example.com?narinfo-compression=gzip&ls-compression=gzip' \
  /nix/store/...
nix store copy-log --to \
  'http://cache.example.com?log-compression=br' \
  /nix/store/...
```

HTTP decompression follows libcurl capabilities (since 2.34.0). Supported
encodings include `deflate`, deprecated `x-gzip`, `br`, `zstd`, and `gzip`;
nonstandard `xz` and `bzip2` are rejected. Distribution builds must link
libcurl to the codec libraries they need.

## S3 caches

### Credentials

The S3 client supports the STS profile credentials provider (since 2.29.0),
including credentials established by `aws sso login`.

Nix 2.34.2 restores container-native providers:

- STS WebIdentity for EKS IRSA, CI OIDC, and other
  `AssumeRoleWithWebIdentity` flows, using `AWS_WEB_IDENTITY_TOKEN_FILE` and
  role variables.
- ECS metadata credentials for ECS tasks and EKS Pod Identity, using
  `AWS_CONTAINER_CREDENTIALS_RELATIVE_URI` or
  `AWS_CONTAINER_CREDENTIALS_FULL_URI`.

### Uploads

S3 cache traffic uses HTTP through curl SigV4 as of 2.33.0. Authenticated
operation requires curl 7.75.0 or later and `aws-crt-cpp`; without
`aws-crt-cpp`, only public buckets are accessible.

Multipart upload settings are:

- `multipart-upload`, default `false`.
- `multipart-threshold`, default 100 MiB.
- `multipart-chunk-size`, default and minimum 5 MiB.
- `buffer-size`, now an alias for `multipart-chunk-size`.

The `storage-class` store parameter applies to both regular and multipart
uploads (since 2.33.0). Omission uses the bucket default:

```sh
nix copy --to 's3://my-bucket?storage-class=INTELLIGENT_TIERING' "$path"
```

S3 addressing defaults to `addressing-style=auto` (since 2.34.0). Standard AWS
endpoints use virtual-hosted URLs; custom endpoints and dotted bucket names
use path style. `path` forces the deprecated path form. `virtual` forces
virtual-hosted addressing and is invalid for dotted bucket names.

```text
s3://my-bucket/key?region=us-east-1&addressing-style=path
```

### Signing

The `secret-keys` store URI parameter accepts a comma-separated key-file list
(since 2.29.0), allowing copied paths to receive multiple signatures during a
key rotation:

```sh
nix copy --to 'file:///tmp/store?secret-keys=/tmp/key1,/tmp/key2' \
  "$(nix build --print-out-paths nixpkgs#hello)"
```

## Cache creation and logging

Nixpkgs `mkBinaryCache` creates zstd-compressed caches by default (since
nixos-25.05). Set `compression = "xz";` to retain the previous output format.

`json-log-path` mirrors every Nix log message as JSON to a file or Unix socket
(since 2.30.0):

```ini
json-log-path = /var/log/nix.json
```

Use a socket when a collector should consume logs continuously, and arrange
rotation and permissions when writing a file.
