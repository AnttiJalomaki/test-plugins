---
name: trivy-knowledge-patch
description: Trivy
version: 0.72.0
license: MIT
metadata:
  author: Nevaberry
---


# Trivy Knowledge Patch

## Use this patch

Load this patch when a task involves Trivy CLI behavior, vulnerability or
license scanning, image and repository acquisition, dependency analysis,
SBOM processing, report consumers, misconfiguration policies, secret scanning,
VEX, embedding Trivy, or client/server mode.

Before changing a command or integration:

1. Identify the scan target and scanners in use: vulnerabilities, licenses,
   misconfigurations, secrets, or SBOM generation and ingestion.
2. Check whether the task consumes Trivy's CLI, Go API, RPC transport, JSON,
   SARIF, CycloneDX, SPDX, or a report template.
3. Treat project configuration, schemas, generated reports, and runtime
   behavior as authoritative when they differ from generic examples.
4. Apply version-dependent migrations before adding new behavior.
5. Open the matching topic reference; the quick reference is deliberately
   selective.

## Reference index

| Reference | Topics |
|---|---|
| [cli-and-runtime.md](references/cli-and-runtime.md) | CLI flags and defaults, configuration, cache behavior, server mode, lifecycle |
| [artifacts-and-registries.md](references/artifacts-and-registries.md) | Image acquisition, registry authentication, archives, layers, repository identity |
| [vulnerabilities-and-os.md](references/vulnerabilities-and-os.md) | Severity sources, OS support, package identity, third-party filtering, detector behavior |
| [dependencies-and-licenses.md](references/dependencies-and-licenses.md) | Language ecosystems, workspaces, lockfiles, dependency graphs, license semantics |
| [sbom-and-reports.md](references/sbom-and-reports.md) | SBOM ingestion and output, report metadata, SARIF, CycloneDX, SPDX, templates |
| [iac-and-misconfigurations.md](references/iac-and-misconfigurations.md) | Terraform/OpenTofu, Kubernetes, Helm, cloud schemas, Rego, Dockerfile and ignore behavior |
| [secrets-and-vex.md](references/secrets-and-vex.md) | Secret detectors and exclusions, VEX loading, TLS, graph handling |

## Breaking migrations and changed defaults

### Migrate provider mappings from `AVDID` to `ID`

Misconfiguration provider mappings use `ID`. Update custom mappings and any
consumer code that still reads or writes `AVDID`; do not support the old field
by silently copying it into the new representation.

### Migrate Docker configuration to `dockers_v2`

The Docker configuration representation is `dockers_v2`. Embedders, custom
configuration adapters, and tests tied to the former representation must move
to the new structure before relying on current Docker scanning behavior.

### Account for all-package listing

`--list-all-pkgs` defaults to `true`. Commands that require the older selective
package output must pass the choice explicitly:

```sh
trivy image --list-all-pkgs=false alpine:3.22
```

### Do not pass filesystem skip flags to `sbom`

The `sbom` command disables `--skip-dir` and `--skip-files`. Remove those flags
from wrappers that invoke `trivy sbom`; scanner-specific exclusions belong in
the supported configuration path.

### Build WebAssembly modules with standard Go

Trivy's WebAssembly modules use standard Go rather than TinyGo. Update compiler
selection and build automation accordingly.

## High-value CLI behavior

### Override OS detection deliberately

Use `--distro` when automatic distribution detection is missing or unsuitable.
If the OS is overwritten, package PURLs are rewritten to match the selected OS,
so compare downstream identities after the override rather than before it.

### Select the vulnerability severity authority

Use `--vuln-severity-source` when severity must come from a particular source:

```sh
trivy image --vuln-severity-source nvd alpine:3.20
```

Missing vulnerability details now fall back to `UNKNOWN` severity. Consumers
must accept that value instead of assuming all findings have a populated source
severity.

### Validate configuration and templates early

Use the JSON Schema for `trivy.yaml` in editors and validation workflows.
The CLI validates `--server` values and template file extensions. Configuration-
only options track whether the user supplied them, so absence must not be
treated as an explicit default.

### Recover rather than trust stale database metadata

When database contents are gone but metadata remains, Trivy downloads the
database again. The vulnerability database also permits concurrent access, so
avoid adding wrapper-level serialization solely to protect normal reads.

## Scan and artifact selection

Registry mirrors and Docker contexts participate in image resolution. Remote
artifact types that are not container images are rejected, and images whose
combined layer size exceeds the configured limit fail early. Preserve these
errors instead of retrying them as ordinary scan failures.

Registry access supports GitHub Container Registry authentication through
`GITHUB_TOKEN`, dual-stack ECR endpoints, and registry login without an
authentication scope. Docker archives retain `RepoTags`.

Filesystem targets with Git information are classified as repository artifacts.
That affects artifact type, sanitized repository metadata, cache hits, SARIF
root paths, and stable artifact/report identities. See the artifact and report
references before keying external storage on a target string.

## Vulnerability and OS scanning

Distribution aliases map to their corresponding OS, and RHEL-derived images can
be recognized from `os-release`. Explicitly supported families include newer
Bottlerocket, MinimOS, AlmaLinux, Root.io, Photon, Ubuntu, Rocky Linux, CoreOS,
ActiveState, Azure/Mariner, Wolfi, and CentOS Stream behaviors documented in the
OS reference.

Debian and Ubuntu packages classified as third-party are skipped by distribution
vulnerability matching. The common detector path applies third-party filtering
too. Preserve repository class across client/server scans and package provenance
through `AnalyzedBy` when a consumer must explain why a package was or was not
matched.

## Dependency and license analysis

Dependency scanning understands modern workspace and lockfile structures,
including uv, Poetry v2, `pylock.toml`, Bun, multi-document pnpm locks, Yarn and
Cargo workspaces, Go binaries built with `-trimpath`, Maven repository settings,
self-contained .NET runtimes, and Julia vulnerability matching.

Do not flatten dependency graphs: Node peer dependencies, workspace roots,
Cargo relationships, `.deps.json` graphs, Julia relationships, and SBOM root
associations all affect result interpretation.

License handling distinguishes SPDX identifiers, full expressions, exceptions,
custom classifications, extracted licensing information, and plain text. Read
the license section before normalizing or filtering values such as `WITH`,
`unlicensed`, non-SPDX text, and `NOASSERTION`.

## SBOM and report consumers

Reports may expose `ArtifactID`, UUIDv7 `ReportID`, vulnerability fingerprints,
image references, repository metadata, analyzer provenance, the Trivy version,
and image layer information. Client/server JSON also reports the server version.
Treat these fields as part of the current result contract.

CycloneDX processing preserves input structure during vulnerability updates,
supports referenced VEX, multiple license shapes, file components, SHA-512,
CVSS v4 ratings, and CycloneDX 1.7. SPDX supports attestations, SHA-512, extracted
licenses, and documents without a root component.

SBOM applications from different files remain distinct, packages from multiple
SBOMs are retained, and OS packages inside and outside dependency graphs are
merged. Do not reintroduce path-only or type-only deduplication that loses these
distinctions.

## Misconfiguration scanning

Terraform/OpenTofu evaluation handles cached remote modules, partial schemas,
unknown values, dynamic blocks, `for_each` maps, policy templates, raw Terraform
data for Rego, and traversal-safe filesystem functions. Parser options and
working-directory context must reach modules as well as roots.

Kubernetes scans support controllers, namespaced components, complete and
summary report semantics, and artifact version comparison without the
`last-applied-configuration` annotation. Helm chart discovery uses exact chart
filenames, includes `.yml`, and applies subdirectory-aware ignore paths.

Ignore handling is context-sensitive: Dockerfile and Helm inline comments are
supported, `.trivyignore` recognizes check aliases, rule identifiers compare
case-insensitively, and an ignore-marker value must be known and non-null.

CloudFormation, AWS, Azure, GCP, GitHub, Ansible, Docker history, and Rego each
have expanded adapters or evaluation rules. Consult the IaC reference before
assuming an unknown resource is unsupported.

## Secrets and VEX

Secret scanning validates UTF-8, reports corrected multiline locations,
supports configurable skipped folders/files/extensions, and has additional
Azure, Maven, Symfony, Hugging Face, OpenAI, and GitHub App token handling. It
skips its own configuration file and Python `.dist-info` directories where
specified.

VEX can be loaded from CycloneDX references and from the scanned repository.
Repository-specific TLS applies per VEX source, and looping package graphs must
not cause vulnerabilities to be suppressed.

## Integration checklist

- Confirm the target's artifact type after Git and registry resolution.
- Pin changed CLI defaults explicitly in automation.
- Preserve package graph, layer, repository, and analyzer metadata in adapters.
- Accept `UNKNOWN` vulnerability severity and multiple valid license shapes.
- Keep client/server fields aligned, including relationships, build information,
  repository class, and server version.
- Exercise IaC ignores against aliases, case variation, chart paths, and unknown
  values.
- Test report consumers with current JSON, SARIF, CycloneDX, SPDX, and templates.
- Recheck the matching reference whenever a task crosses scanner boundaries.
