# SBOM, VEX, and Attestations

## Package and application structure

- Nested packages stay attached to their application. Applications of the same
  type from different SBOM files remain distinct, image layer data is included
  in SBOM scan results, and unknown dependencies attach to a root package when
  one exists (since 0.59.0).
- If an application path cannot otherwise be determined, the SBOM file path is
  used as its `FilePath` (since 0.60.0).
- OS packages from multiple SBOM inputs are preserved (since 0.60.0).
- OS packages found inside and outside an SBOM dependency graph are merged
  (since 0.65.0).
- CoreOS SBOM scanning is supported (since 0.67.0).

## CycloneDX

- External VEX documents referenced from a CycloneDX SBOM can be loaded (since
  0.60.0).
- Tool metadata includes `manufacturer` (since 0.64.0).
- SHA-512 component hashes are supported, and generated reports place licenses
  in the correct CycloneDX field (since 0.65.0).
- Components may carry multiple license variants, and `file` components are
  accepted (since 0.66.0).
- Vulnerability updates to a CycloneDX file preserve the input document's
  structure (since 0.67.0).
- SBOM output exposes `buildInfo` through properties. Client/server RPC carries
  the equivalent data in `BlobInfo` (since 0.68.0).
- CycloneDX vulnerability output supports CVSS v4 ratings (since 0.70.0).
- CycloneDX 1.7 is supported (since 0.71.0).

## SPDX

- Non-standard licenses are emitted through `hasExtractedLicensingInfos`
  (since 0.59.0).
- Text licenses retain their unnormalized text in `otherLicenses` (since
  0.61.0).
- Image/SBOM handling accepts SPDX attestations (since 0.68.0).
- For non-library packages, SPDX output sets `licenseDeclared` and
  `licenseConcluded` to `NOASSERTION` (since 0.70.0).
- SPDX serialization supports SHA-512 hashes (since 0.71.0).
- The SPDX marshaler tolerates a document with no root component (since
  0.72.0).

## VEX

- A package involved in a looping dependency graph is not incorrectly
  suppressed by VEX processing (since 0.67.0).
- VEX repositories support independent TLS configuration per repository (since
  0.69.0).
- VEX documents stored within the scanned repository directory are discovered
  (since 0.72.0).

## Embedded and attested SBOMs

- Docker archives retain `RepoTags`, while SBOMs in Sigstore bundles and SPDX
  attestations are recognized (since 0.68.0).
- Images containing embedded SBOMs produce deterministic results (since
  0.69.0).
- Red Hat `BuildInfo` survives SBOM scanning even without layer information
  (since 0.70.0).
- Client/server scanning preserves each package's repository class across RPC
  transport (since 0.72.0).

