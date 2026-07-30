# Vulnerabilities and Operating Systems

## Severity and matching controls

- `--vuln-severity-source` selects the vulnerability severity source
  (since 0.60.0).
- NuGet package names compare case-insensitively during vulnerability analysis,
  preventing misses caused only by casing (since 0.67.0).
- VEX processing does not suppress vulnerabilities merely because the affected
  package participates in a looping dependency graph (since 0.67.0).
- Vulnerability results without details use `UNKNOWN` severity rather than
  inventing a populated severity (since 0.72.0).
- `ospkg.NewScanner` forwards detector options to OS package detectors, so
  embedding callers can rely on those options taking effect (since 0.72.0).

## OS identification and explicit overrides

- `--distro` manually selects the distribution when automatic detection is
  unavailable or inappropriate (since 0.59.0).
- Distribution aliases map to their corresponding operating systems
  (since 0.60.0).
- RHEL-derived images can be detected from `os-release` data (since 0.67.0).
- When the detected OS is overwritten, OS package PURLs are updated to match the
  replacement OS (since 0.71.0).

An override changes package identity as well as detector selection. Any consumer
that caches by PURL should evaluate the post-override identity.

## Distribution and image coverage

- A Bottlerocket OS package analyzer and MinimOS support are available
  (since 0.63.0). Bottlerocket packages can be matched against vulnerability
  data (since 0.72.0).
- Wolfi scanning recognizes the newer APK database location (since 0.63.0).
- Root.io container images are supported (since 0.64.0); package detection uses
  the full package version and findings use corrected severity selection
  (since 0.65.0).
- AlmaLinux 10 is supported, and the Alma `rpmqa` parser accepts epoch-qualified
  versions (since 0.65.0).
- CoreOS SBOMs are supported (since 0.67.0).
- Photon 5.0 vulnerability scanning is supported (since 0.68.0).
- ActiveState image scanning is supported (since 0.69.0).
- Rocky Linux modular packages participate in vulnerability detection
  (since 0.69.0).
- For CentOS Stream, unsupported vulnerability detection is skipped without
  failing the overall scan (since 0.69.0).
- Ubuntu 26.04 LTS is recognized by OS detection (since 0.71.0).

## Lifecycle data

- Lifecycle data includes EOL dates for RHEL 10, Ubuntu 25.04, and Ubuntu 20.04
  ESM (since 0.64.0).
- Ubuntu 25.10 EOL data is included (since 0.70.0).

Keep lifecycle/EOL decisions separate from vulnerability matching. A detected
and supported OS can still be out of lifecycle, and a family can be recognized
while a particular vulnerability detector is intentionally skipped.

## Third-party packages and package metadata

- Debian packages are classified as third-party from a maintainer list. Debian
  and Ubuntu vulnerability scans skip those packages rather than applying
  distribution advisories to them (since 0.69.0).
- The common vulnerability path also skips packages marked third-party
  (since 0.70.0).
- Red Hat package analysis searches the root layer for build information,
  retains `contentSets` for OS packages in filesystem and VM scans, and trims
  invalid suffixes from manifest `content_sets` (since 0.63.0).
- SBOM scans preserve Red Hat `BuildInfo` even when layer data is absent
  (since 0.70.0).
- Alpine APK analysis exposes the package maintainer (since 0.63.0).
- Azure and Mariner detectors populate detected-vulnerability fields in their
  results (since 0.70.0).
- Package results expose `AnalyzedBy` to identify the analyzer that detected the
  package (since 0.69.0).
- Client/server transport preserves each package's repository class
  (since 0.72.0).

## Ecosystem-specific vulnerability identity

- Julia packages participate in vulnerability matching (since 0.69.0).
- Go pseudo-versions use the linker-flags version for every pseudo-version when
  build metadata supplies it (since 0.69.0).
- Go binaries built with `-trimpath` can derive versions from the ELF symbol
  table, and Go 1.26 `GOEXPERIMENT` version formatting is understood
  (since 0.70.0).
- npm constraint comparison no longer applies prerelease logic
  (since 0.65.0).
