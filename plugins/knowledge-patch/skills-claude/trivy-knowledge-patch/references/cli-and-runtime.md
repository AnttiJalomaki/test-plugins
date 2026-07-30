# CLI and Runtime

## Scanner and report controls

- Vulnerability scans accept `--distro` to override automatic OS detection
  when it is unavailable or unsuitable (since 0.59.0).
- `--generate-default-config` omits hidden flags from generated configuration
  (since 0.59.0).
- Use `--vuln-severity-source` to choose the authority for vulnerability
  severities (since 0.60.0).
- Reports can include a summary table for a compact overview (since 0.60.0).
- Kubernetes scans support `--report all`, and the requested complete report is
  emitted rather than being lost (since 0.61.0). Kubernetes summary reports
  omit passed misconfigurations (since 0.62.0).
- The CLI no longer constrains `--compliance` to a fixed allowed-values list
  (since 0.63.0).
- Trivy can check whether a newer Trivy version is available (since 0.63.0).
- The `sbom` command disables `--skip-dir` and `--skip-files`; remove these
  arguments from command wrappers (since 0.63.0).
- `--server` is schema-validated, so malformed server values fail during flag
  validation (since 0.65.0).
- `--list-all-pkgs` defaults to `true`; use `--list-all-pkgs=false` to retain
  selective package output (since 0.67.0).
- `trivy cloud` is available, and `--cacert` supplies a CA certificate
  (since 0.68.0).
- Template flag validation checks file extensions and rejects invalid template
  files (since 0.70.0).

## Configuration semantics

- Check metadata accepts an `examples` field (since 0.59.0).
- Configuration-only flags record whether the user actually supplied them;
  their defaults no longer masquerade as explicit user choices (since 0.68.0).
- Boolean values in misconfiguration check metadata are interpreted as booleans
  (since 0.68.0).
- A JSON Schema for `trivy.yaml` supports editor assistance and validation;
  enum constraints for array values live on the schema's item definition
  (since 0.69.0).
- Docker configuration uses `dockers_v2`; this is a breaking representation
  migration for embedders and configuration consumers (since 0.72.0).

## Process lifecycle and transport

- Termination signals trigger graceful shutdown, while a normal exit does not
  print the graceful-shutdown message (since 0.65.0).
- HTTP request/response tracing is supported, and server mode configures an HTTP
  transport (since 0.65.0).
- JSON reports include the producing Trivy version (since 0.69.0).
- In client/server mode JSON reports also include the server version. The
  server's `/version` response omits JavaDB and CheckBundle entries
  (since 0.70.0).
- Client/server RPC transports package dependency `Relationship` values
  (since 0.63.0), SBOM `buildInfo` through `BlobInfo` (since 0.68.0), and package
  repository class (since 0.72.0).

## Cache, database, and plugins

- If database files are missing while metadata remains, Trivy downloads the
  database again rather than accepting stale metadata as a valid cache
  (since 0.67.0).
- The vulnerability database permits concurrent access by multiple scans
  (since 0.68.0).
- Filesystem report cache hits preserve `RepoMetadata` (since 0.66.0).
- Updating plugin `index.yaml` preserves existing plugins instead of removing
  them (since 0.66.0).

## Report-mode operational checks

When maintaining automation, explicitly pin changed defaults, allow validation
errors to reach the caller, and distinguish normal process exit from a
signal-driven graceful shutdown. In distributed deployments, verify that the
client, server, and report consumer all accept the expanded metadata rather
than stripping it during serialization.
