# Release Compatibility and Upgrade Boundaries

## Select stable builds

- The Android v1.82.0 client did not ship on the normal schedule. Android's
  stable build was v1.82.1; v1.82.1 and v1.82.4 were Android-only, while
  v1.82.2 and v1.82.3 were internal-only (1.82.0).
- The v1.86.0 rollout was halted for macOS on July 25, 2025, and for all
  platforms on July 28 because of several regressions. Versions 1.86.1 and
  1.86.3 were internal-only. Version 1.86.2 fixes a macOS state-file read
  failure that could require device re-approval. Standalone macOS v1.86.4
  fixes a fresh-install startup crash when `EncryptState` is enabled (1.86.0).
- Version 1.88.0 was internal-only. QNAP distribution resumed first as a manual
  package-site download, with QNAP App Center availability following later
  (1.88.1).
- Version 1.90.0 was a release candidate for testing; v1.90.1 is the stable
  release (1.90.1).
- Version 1.92.0 was a release candidate for testing; v1.92.1 is the stable
  release (1.92.1).
- Version 1.94.0 was a release candidate for testing; v1.94.1 is the stable
  release (1.94.1).
- Versions 1.96.0 and 1.96.1 were release candidates for testing; v1.96.2 is
  the stable release (1.96.2).
- Version 1.98.0 was a release candidate for testing. The Linux v1.98.1 build
  was withdrawn because of a regression in its interaction with MagicDNS;
  wait for a corrected Linux build rather than deploying it (1.98.1).

## Account for platform requirements and distribution details

- macOS 12 is the minimum supported macOS version (1.88.1).
- Windows v1.84.2 rotates to a code-signing certificate with the same subject
  and issuer as its predecessor but a different serial number. Update any
  deployment control that allowlists the signing certificate by serial
  (1.84.0).
- The macOS About screen can select the release-candidate channel and keep
  release-candidate builds automatically updated. Use this only for devices
  intended to test that channel (1.94.1).
- On OpenWrt 25.12.0 and later, Tailscale can update itself when `apk` is the
  package manager (1.96.2).

## Apply regression fixes before diagnosing configuration

- The strict duplicate-flag parsing introduced in v1.84.0 also prevented the
  container's `TS_EXTRA_ARGS` from setting `--accept-dns`. Container image
  v1.84.2 restores that supported use (1.84.0).
- Version 1.84.1 corrects an unintended default-off state for subnet routing on
  iOS and Android. Do not infer the desired managed setting from the affected
  default (1.84.0).
- The v1.86 stable line fixes a CSRF problem that could break web-interface
  login and restores hostname verification for control-plane connections sent
  through a CONNECT HTTPS proxy. It also improves Windows 10 version 1607 and
  earlier proxy auto-detection and PAC handling (1.86.0).
- For Standalone macOS encrypted-state deployments, v1.86.4 also applies
  `EncryptState` policy changes without restarting the system extension
  (1.86.0).
