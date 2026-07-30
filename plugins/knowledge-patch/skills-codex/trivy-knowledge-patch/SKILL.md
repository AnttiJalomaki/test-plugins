---
name: trivy-knowledge-patch
description: Trivy
version: 0.72.0
license: MIT
metadata:
  author: Nevaberry
---


# Trivy Knowledge Patch

Use this skill when configuring, invoking, embedding, or consuming results from
Trivy. Start with the quick references below, then load the topic reference that
matches the work at hand.

## Reference index

| Reference | Topics |
| --- | --- |
| [CLI, reports, and runtime](references/cli-reports-and-runtime.md) | CLI defaults and validation, configuration, report identity and formats, server mode, databases, plugins, cloud scans |
| [Images, registries, and operating systems](references/images-registries-and-os.md) | Image acquisition and history, registry behavior, OS detection, package metadata, lifecycle and vulnerability matching |
| [Dependencies and licenses](references/dependencies-and-licenses.md) | Language package analyzers, lockfiles, workspace graphs, Maven behavior, license discovery and classification |
| [SBOM, VEX, and attestations](references/sbom-vex-and-attestations.md) | CycloneDX and SPDX handling, VEX, embedded and attested SBOMs, dependency graphs and transport metadata |
| [Misconfiguration and IaC](references/misconfiguration-and-iac.md) | Terraform/OpenTofu, CloudFormation, Kubernetes, Helm, Azure/GCP/AWS schemas, Rego, ignores, image checks |
| [Secrets and filtering](references/secrets-and-filtering.md) | Secret inspection, exclusions, detectors, input validation, locations, and false-positive controls |

## Breaking changes and migration checks

### Migrate provider mappings

Misconfiguration provider mappings use `ID` rather than `AVDID` (since
0.69.0). Update custom mapping structures and every consumer that reads the old
field before upgrading.

### Migrate Docker configuration consumers

Docker configuration uses the `dockers_v2` representation (since 0.72.0).
Update integrations that deserialize or inspect the earlier representation.

### Decide whether all packages should be listed

`--list-all-pkgs` defaults to `true` (since 0.67.0). Set the old behavior
explicitly when smaller, selective output is required:

```sh
trivy image --list-all-pkgs=false alpine:3.22
```

### Remove unsupported SBOM skip flags

The `sbom` command disables `--skip-dir` and `--skip-files` (since 0.63.0).
Remove those flags from automation rather than expecting them to filter SBOM
generation.

### Expect early input rejection

Remote image retrieval rejects unsupported artifact types (since 0.64.0), and
image scans reject inputs whose total layer size exceeds the configured limit
(since 0.59.0). Treat both as input errors and surface them without retrying the
same artifact unchanged.

## High-value CLI behavior

### Select the distribution manually

Use `--distro` when automatic OS detection is missing or inappropriate. When
overriding the OS, expect package PURLs to be rewritten to the selected OS
identity.

### Control vulnerability severity provenance

Choose the severity source with `--vuln-severity-source`:

```sh
trivy image --vuln-severity-source nvd alpine:3.20
```

### Use custom trust for cloud scans

Use `trivy cloud` for cloud scanning and `--cacert` when the endpoint requires a
custom CA certificate.

### Validate generated configuration assumptions

`--generate-default-config` excludes hidden flags. Configuration-only options
track whether the user actually supplied them, so a default value is not
equivalent to explicit configuration.

### Handle databases as shared runtime state

The vulnerability database supports concurrent access. If contents disappear
but metadata remains, Trivy downloads the database again; stale metadata alone
is not a valid cache.

## High-value result behavior

### Preserve stable identities

Targets expose an `ArtifactID` derived partly from registry and repository.
Reports expose a UUIDv7 `ReportID`; vulnerability results carry fingerprints,
and report metadata includes the image reference. Use these fields rather than
inventing identity from display text.

### Track analyzer provenance

Detected packages expose `AnalyzedBy`. Preserve it when transforming reports so
consumers can identify which analyzer produced a package.

### Consume version metadata

JSON reports include the producing Trivy version. Client/server JSON reports
also include the server version, while the server `/version` response omits
JavaDB and CheckBundle entries.

### Account for repository scans

Filesystem scans become repository artifacts when Git metadata is detected.
Repository metadata is preserved through filesystem-cache hits, its URL is
sanitized before reporting, and SARIF uses the repository-aware `ROOTPATH` URI.

## High-value SBOM and VEX behavior

### Keep the source document structure

Updating vulnerabilities in a CycloneDX file preserves its structure. Trivy
also preserves applications of the same type from different SBOM files and OS
packages collected from multiple SBOM inputs.

### Load VEX from all supported locations

CycloneDX documents can reference external VEX files. VEX can also be loaded
from the scanned repository, and each VEX repository can have its own TLS
configuration.

### Preserve graph semantics

Nested packages remain attached to their application, unknown dependencies can
be associated with a root package, and OS packages from both inside and outside
an SBOM dependency graph are merged. Do not flatten these relationships during
post-processing.

### Accept modern and attested formats

CycloneDX 1.7, Sigstore-bundle SBOMs, SPDX attestations, and SPDX documents
without a root component are valid inputs. Docker archives retain `RepoTags`.

## High-value dependency analysis

### Respect workspace and graph context

Yarn and Cargo analysis records root/workspace packages and relationships.
Node.js peer dependencies affect the resulting tree, `.NET` uses dependency
graphs from `.deps.json`, and pnpm snapshot strings form package IDs.

### Configure Maven resolution completely

Maven analysis expands all environment placeholders and reads repositories,
proxies, and mirrors from `settings.xml`. Parent POM properties and repositories
flow into dependencies. An HTTP 429 from a remote repository is fatal because
continuing would yield incomplete resolution.

### Use current Python inputs

Python analysis supports uv dependency groups, Poetry v2, `.egg-info/METADATA`,
and PEP 751 `pylock.toml`. PEP 770 files below `.dist-info/sboms/` are excluded
from ordinary SBOM discovery.

### Preserve license expressions

Treat SPDX identifiers, compound expressions, and `WITH` exceptions
semantically. Do not normalize the literal `unlicensed` to `Unlicense`, and do
not split an SPDX exception from its license during category detection.

## High-value misconfiguration behavior

### Treat unknown Terraform values as unknown

Missing variables become unknown values. Dynamic blocks expand only when
`for_each` is known, while map-valued `for_each` creates one resource per key.
Terraform filesystem functions prevent path traversal.

### Apply parser options consistently

Terraform parser options apply to submodules, including remote modules cached
under `.terraform`; cached submodules retain their original paths. A parser
working directory can be set explicitly.

### Modernize custom Rego integration

The IaC scanner accepts an injected Rego scanner. Policies can consume raw
Terraform data, ignore finding types, and use a configurable error limit.
Manifest diagnostic snippets include map keys.

### Apply ignores predictably

Inline ignores work for Dockerfiles and Helm content. Chart-subdirectory paths
are respected, identifiers match case-insensitively, and `.trivyignore` applies
check aliases. An ignore-marker expression only suppresses a result when its
value is known and non-null.

### Prefer parsed effective image state

For image misconfiguration, `.Config.User` wins over `USER` history entries.
Buildah, legacy-builder, non-BuildKit, and build-metadata history forms are
normalized or reconstructed before instruction checks run.

## High-value secret scanning

### Keep client/server scans equivalent

The configuration analyzer performs secret inspection in client/server mode.
Repository class and other package metadata should survive RPC transport.

### Customize exclusions

Skipped secret-scan folders, files, and extensions are configurable. Python
`.dist-info` directories and the secret-scanner configuration file itself are
excluded from inspection.

### Preserve safe input and locations

Secret input is validated as UTF-8 before protobuf marshalling. Multiline secret
line numbers are corrected, so downstream presentation should use the reported
locations directly.

## Working method

1. Identify the scan target and mode: image, filesystem, repository, SBOM,
   Kubernetes, cloud, or client/server.
2. Check the relevant CLI defaults and breaking migrations before composing
   flags or configuration.
3. Load the matching topic reference and retain its version-sensitive behavior
   in code, templates, and report consumers.
4. Preserve IDs, graph relationships, provenance, paths, and transport metadata
   unless a downstream contract explicitly requires a lossy transformation.
5. Validate automation against expected failure behavior, especially input
   rejection, template validation, Maven throttling, and unknown IaC values.
