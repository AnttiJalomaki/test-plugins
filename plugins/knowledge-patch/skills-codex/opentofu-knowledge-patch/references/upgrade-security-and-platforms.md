# Upgrade, security, and platform boundaries

Check these constraints before changing the required OpenTofu version or base
runtime.

## Supported release floor

OpenTofu 1.6 is unsupported from 1.9.0 and receives no further security
updates. Upgrade to at least 1.7.

## Container-image changes

Using `ghcr.io/opentofu/opentofu` as the base for custom images is deprecated
in 1.9 and unsupported in 1.10. Change custom-image builds before upgrading.

## Operating-system requirements

OpenTofu 1.10.0 requires Linux kernel 3.2+ or macOS 11+.

OpenTofu 1.11.0 requires macOS 12 Monterey or later. The 1.12.0 series is the
last planned line to support macOS 12.

Official 32-bit `386` and `arm` packages continue through 1.13 but are planned
for later removal. `amd64` and `arm64` are unaffected.

## Windows junction behavior

Since 1.10, Windows junctions are not treated as symbolic links. A `TEMP` path
that traverses a junction may fail; use a real directory symbolic link
instead.

## Transport hardening

Since 1.11, TLS handshakes reject SHA-1 signatures. SSH also rejects
incorrectly generated certificates whose signing key is itself a certificate
key.

Use 1.11.4 or later when provider or module installation may encounter an
untrusted ZIP archive. Earlier 1.11 releases can spend excessive time
processing a malicious archive.

## Provisioner deprecation

WinRM connections work in 1.12 but warn and are planned to become errors in
1.13. Migrate Windows provisioner targets to SSH.

## Patch-level operational fixes

- Use 1.10.2+ for native S3 lockfiles when the bucket requires server-side
  encryption.
- Use 1.10.5+ when processes share `TF_PLUGIN_CACHE_DIR`.
- Use 1.11.4+ for untrusted provider or module ZIP archives and when modules
  with local provider configurations interact with `lifecycle.enabled`.
- Use 1.12.4+ when saving a plan that may replace a resource configured with
  `lifecycle.destroy = false`.

## OpenTofu 1.12 security floor

Do not remain on 1.12.0 or another early 1.12 patch. Early releases had
security defects involving SSH connections, OpenBao-wrapped state-encryption
data, revoked SSH CA keys, and malicious Git URLs that could read arbitrary
files. Use the latest 1.12 patch available; that is 1.12.4 in this guidance.

## Sensitive logs and explicit disclosure

HTTP backend trace logs include request and response bodies from 1.9. Protect
them as state-bearing output.

`-show-sensitive` deliberately unmasks protected values in commands returning
configuration or state. Avoid it in shared terminals and captured CI logs.

Experimental external encryption programs receive keys or plaintext payloads,
and initialization tracing sends data to a collector. Run both only under the
operator's control and evaluate their data path before use.
