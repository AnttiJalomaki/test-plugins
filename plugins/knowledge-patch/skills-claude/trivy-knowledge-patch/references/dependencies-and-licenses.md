# Dependencies and Licenses

## Node.js, Bun, and pnpm

- Node.js dependency trees include peer-dependency relationships
  (since 0.59.0).
- Yarn analysis records root/workspace context for packages (since 0.62.0).
- `bun.lock` is parsed and analyzed (since 0.63.0), including its `packages`
  array layout (since 0.64.0).
- npm constraint comparison avoids prerelease logic (since 0.65.0).
- `package-lock.json` accepts object-form workspace declarations. pnpm uses the
  snapshot string as `Package.ID` (since 0.67.0).
- `package-lock.json` license data is read into package results (since 0.69.0).
- Legacy npm lockfile license formats are supported. A subdirectory
  `package.json` with an invalid name is skipped instead of aborting analysis
  (since 0.71.0).
- Project dependencies are parsed from multi-document `pnpm-lock.yaml` files
  (since 0.72.0).

## Python

- uv projects are supported, including dev and optional dependencies
  (since 0.59.0).
- Poetry dev dependencies are supported, while dependencies in Poetry's dev
  group are skipped (since 0.59.0). Poetry v2 projects are supported
  (since 0.60.0).
- Package metadata in `.egg-info/METADATA` is recognized (since 0.65.0).
- PEP 770 SBOM files under `.dist-info/sboms/` are excluded from normal SBOM
  discovery (since 0.69.0).
- PEP 751 `pylock.toml` lock files are parsed and scanned (since 0.70.0).
- `requirements.txt` accepts dependencies with multiple version specifiers.
  Empty optional Poetry groups no longer crash analysis (since 0.70.0).

## Go

- The main-module version in artifacts produced by Go 1.24 or newer is parsed
  correctly (since 0.60.0).
- Echo is supported (since 0.63.0).
- License discovery searches dependencies in `GOPATH` and `vendor`; `go.mod`
  projects also search the vendor tree for license files (since 0.63.0).
- Linker-flags versions are used for all pseudo-versions when build metadata is
  present (since 0.69.0).
- `-trimpath` binaries can obtain versions from the ELF symbol table, and the
  Go 1.26 `GOEXPERIMENT` version form is supported (since 0.70.0).
- WebAssembly modules build with standard Go rather than TinyGo
  (since 0.61.0).

## Maven and Java

- Every environment placeholder in Maven `settings.xml` is dereferenced, not
  just the first one (since 0.64.0).
- Repositories are read from `settings.xml`; releases and snapshots default to
  enabled when omitted. Fields from multiple dependency-management sources use
  corrected precedence (since 0.68.0).
- POM properties inherit from parent fields, and repositories from upper POMs
  propagate to dependencies. `pom.xml` package IDs include a hash of GAV
  coordinates and the root-POM path to avoid collisions (since 0.69.0).
- Maven proxy configuration in `settings.xml` controls repository access, and
  Java dependency exclusions are preserved rather than overwritten
  (since 0.70.0).
- Maven `<mirrors>` are honored. An HTTP 429 from a remote repository while
  scanning a POM is fatal, avoiding silently incomplete resolution
  (since 0.71.0).
- JAR license discovery checks packaged `LICENSE` files and embedded `pom.xml`
  files (since 0.72.0).
- Gradle lockfile analysis excludes development dependencies (since 0.63.0).

## Rust, Julia, .NET, Composer, and Seal

- Cargo lockfile analysis records root/workspace packages and relationships
  (since 0.62.0).
- Cargo monorepos expand globbed workspace members and support inherited package
  versions (since 0.69.0).
- Julia parsing populates `Relationship`, and RPC preserves it (since 0.63.0).
  Julia packages can be vulnerability-scanned (since 0.69.0).
- `.deps.json` analysis builds .NET dependency graphs rather than emitting an
  unconnected package list (since 0.68.0).
- Self-contained .NET deployments expose their bundled runtime as components
  (since 0.72.0).
- Composer analysis includes development dependencies (since 0.69.0).
- Seal is supported natively (since 0.67.0), and Seal language-file detection
  understands vendor information (since 0.71.0).

## License discovery and classification

- Non-SPDX licenses are represented with `hasExtractedLicensingInfos`
  (since 0.59.0).
- Debian `dpkg` results omit empty license values. SPDX text licenses are stored
  unchanged in `otherLicenses` (since 0.61.0).
- Configured custom classifications apply to text licenses, and compound
  expressions using SPDX operators are supported (since 0.63.0).
- License scans honor the package-types selection (since 0.65.0).
- `GFDL-NIV-1.1` and `GFDL-NIV-1.2` are mapped, and `LaxSplitLicenses` handles
  the `WITH` operator (since 0.65.0).
- CycloneDX licenses are emitted in the correct field (since 0.65.0), including
  components with multiple license types (since 0.66.0).
- License analysis validates identifiers against the SPDX list and separates
  SPDX IDs from full expressions when applying ignores. A `WITH` exception
  remains attached to its license during category detection, and literal
  `unlicensed` is not normalized to `Unlicense` (since 0.68.0).
- Output uses canonical identifiers from embedded SPDX license data
  (since 0.69.0).
- SPDX emits `licenseDeclared` and `licenseConcluded` as `NOASSERTION` for
  non-library packages (since 0.70.0).
