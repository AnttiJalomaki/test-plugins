# CLI and Release Automation

## Configuration and CI

- `tailscale configure` and its subcommands are no longer alpha as of 1.80.0,
  except for `tailscale configure kubeconfig`. The Standalone macOS client adds
  `tailscale configure sysext activate`, `deactivate`, and `status` for
  programmatic system-extension management.
- The Tailscale GitHub Action is generally available on macOS and Windows
  runners in 1.82.0-era releases. Set `use-cache: 'true'` to cache Tailscale
  binaries.
- Linux can install a freedesktop autostart entry for the tray application
  (since 1.96.2):

  ```console
  tailscale configure systray --enable-startup=freedesktop
  ```

## Writing unattended commands

- Starting with 1.84.0, CLI commands reject multiple occurrences of the same
  flag. The stricter parser initially stopped the container's `TS_EXTRA_ARGS`
  from setting `--accept-dns`; container image 1.84.2 restores that use.
- Starting with 1.88.1, significant actions can ask for `y/n` confirmation.
  Audit scripts and subprocess integrations for commands that can now become
  interactive.
- `tailscale dns query` and `tailscale dns status` accept `--json` for
  machine-readable output (since 1.96.2):

  ```console
  tailscale dns status --json
  ```

- `tailscale wait [flags]` waits for Tailscale resources to become available
  for binding. `tailscale ip --assert=<specific-ip-address>` verifies that an
  address matches at least one Tailscale IP on the node (since 1.96.2):

  ```console
  tailscale wait
  tailscale ip --assert=100.64.0.1
  ```

- The `release-candidate` track is accepted by both `tailscale version
  --track` and `tailscale update --track` (since 1.96.2):

  ```console
  tailscale version --track=release-candidate
  tailscale update --track=release-candidate
  ```

## Platform and stable-line availability

- In the 1.82.0 line, the Android build was delayed until 1.82.1. Releases
  1.82.1 and 1.82.4 are Android-only; 1.82.2 and 1.82.3 were internal-only.
- The 1.86.0 rollout stopped for macOS on July 25, 2025, and for all platforms
  on July 28 because of regressions; 1.86.1 and 1.86.3 were internal-only.
  Version 1.86.2 fixes a macOS state-file read failure that could require
  device re-approval. Version 1.86.4 fixes a fresh-install crash in the
  Standalone macOS client when `EncryptState` is enabled.
- Version 1.88.0 was internal-only; 1.88.1 is the applicable public line. QNAP
  builds resumed first as manual package-site downloads and later through
  QNAP App Center.
- Version 1.90.0 was a release candidate intended only for testing; 1.90.1 is
  the stable release.
- Version 1.92.0 was a release candidate intended only for testing; 1.92.1 is
  the stable release.
- Version 1.94.0 was a release candidate intended only for testing; 1.94.1 is
  the stable release.
- Versions 1.96.0 and 1.96.1 were release candidates intended only for
  testing; 1.96.2 is the stable release.
- Version 1.98.0 was a release candidate intended only for testing. Linux
  1.98.1 was withdrawn because of a regression in its interaction with
  MagicDNS, pending a fix.
