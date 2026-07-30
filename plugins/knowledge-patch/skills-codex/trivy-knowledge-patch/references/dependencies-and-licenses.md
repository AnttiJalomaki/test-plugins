# Dependencies and Licenses

## Node.js and JavaScript ecosystems

- Node.js dependency trees account for peer dependencies, so peer relationships
  can change the dependency graph in results (since 0.59.0).
- Yarn analysis records root/workspace context for packages (since 0.62.0).
- `bun.lock` files are parsed and analyzed (since 0.63.0), including their
  `packages` array (since 0.64.0).
- npm constraint comparison no longer applies prerelease logic (since 0.65.0).
- Object-form workspace declarations in `package-lock.json` are accepted. pnpm
  package IDs use the snapshot string (since 0.67.0).
- Node.js analysis reads license metadata from `package-lock.json` (since
  0.69.0).
- Legacy npm lockfile license formats are accepted. Invalid package names in
  subdirectory `package.json` files are skipped without aborting analysis
  (since 0.71.0).
- Project dependencies are parsed from multi-document `pnpm-lock.yaml` files
  (since 0.72.0).

## Python dependencies

- uv projects are supported, including uv development and optional
  dependencies. Poetry development dependencies are supported, but
  dependencies in Poetry's `dev` group are skipped (since 0.59.0).
- Poetry v2 projects are supported (since 0.60.0).
- Package metadata in `.egg-info/METADATA` is recognized (since 0.65.0).
- PEP 770 SBOM files under `.dist-info/sboms/` are excluded from normal SBOM
  discovery (since 0.69.0).
- PEP 751 `pylock.toml` lock files are parsed and scanned (since 0.70.0).
- `requirements.txt` accepts dependencies with multiple version specifiers.
  Empty optional Poetry groups no longer cause a crash (since 0.70.0).

## Go, Rust, Julia, and .NET

- Go artifacts produced with Go 1.24 or later yield the correct main-module
  version (since 0.60.0).
- WebAssembly modules use standard Go rather than TinyGo, which changes the
  compiler required to build them (since 0.61.0).
- Cargo lockfile analysis records root/workspace packages and their
  relationships (since 0.62.0).
- Julia parsing populates `Relationship`, and client/server RPC transports that
  field (since 0.63.0).
- `.NET` dependency analysis builds graphs from `.deps.json` rather than
  representing packages as an unconnected list (since 0.68.0).
- Julia packages participate in vulnerability matching (since 0.69.0).
- Go pseudo-versions use the linker-flags version consistently, which can alter
  the version reported and matched when build metadata provides it (since
  0.69.0).
- Cargo monorepos expand workspace-member globs and support inherited package
  versions (since 0.69.0).
- For `-trimpath` binaries, Go versions can be recovered from the ELF symbol
  table. Go 1.26 `GOEXPERIMENT` version formatting is also handled (since
  0.70.0).
- Self-contained `.NET` deployments expose their bundled runtime as components
  (since 0.72.0).

## Java and Maven resolution

- Gradle lockfile analysis excludes development dependencies (since 0.63.0).
- Every environment-variable placeholder in Maven `settings.xml` is expanded,
  including files containing multiple placeholders (since 0.64.0).
- Remote repositories are read from Maven `settings.xml`. Releases and
  snapshots default to enabled when those settings omit a value, and fields
  from multiple dependency-management sources follow corrected precedence
  (since 0.68.0).
- POM properties inherit from parent fields, and repositories from upper POMs
  propagate to dependencies. `pom.xml` package IDs include a hash of the GAV
  coordinates and root-POM path to avoid collisions (since 0.69.0).
- Maven proxy settings are used for repository access (since 0.70.0).
- Java dependency exclusions are preserved rather than overwritten (since
  0.70.0).
- Maven `<mirrors>` in `settings.xml` are honored. HTTP 429 from a remote Maven
  repository while scanning a POM is fatal rather than producing incomplete
  resolution (since 0.71.0).

## Other ecosystem detection

- Echo is supported (since 0.63.0).
- Native Seal support is available (since 0.67.0), and Seal language-file
  detection includes vendor information (since 0.71.0).
- NuGet package-name matching is case-insensitive (since 0.67.0).
- Composer analysis includes development dependencies (since 0.69.0).

## License discovery

- Go license scanning searches both `GOPATH` and `vendor`. For `go.mod`
  projects it also searches the vendor directory (since 0.63.0).
- Package-type selection applies to license results (since 0.65.0).
- JAR license discovery reads packaged `LICENSE` files and embedded `pom.xml`
  content (since 0.72.0).

## License classification and expressions

- SPDX text licenses retain their original text in `otherLicenses`, without
  normalization (since 0.61.0).
- Configured custom classifications apply to text licenses. Compound
  expressions using SPDX operators are supported (since 0.63.0).
- `GFDL-NIV-1.1` and `GFDL-NIV-1.2` are mapped, and `LaxSplitLicenses` handles
  the `WITH` operator (since 0.65.0).
- License identifiers are checked against the SPDX ID list. Ignore processing
  distinguishes individual SPDX IDs from full expressions; `WITH` exceptions
  remain attached during category detection, and literal `unlicensed` is not
  normalized to `Unlicense` (since 0.68.0).
- Output uses canonical SPDX identifiers from the embedded license data (since
  0.69.0).

