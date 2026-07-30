# PCL and conversion

This reference includes `3.145.0-3.159.0`, `3.160.0-3.181.0`,
`3.214.1-3.228.0`, `3.229.0-3.248.0`, and `3.249.0-3.254.0`.

## Convert programs

`pulumi convert --from=<plugin>@<version>` pins a converter plugin. Conversion
can bridge Terraform providers automatically, and PCL generation understands
`try` and `can` (`3.145.0-3.159.0`).

```shell
pulumi convert --from=terraform@1.2.3
```

Third-party conversion sources resolve provider plugins through the Pulumi
Registry (`3.229.0-3.248.0`).

## HCL runtime and converter

The HCL language runtime is downloaded on demand rather than bundled.
`pulumi convert --from hcl` installs its converter automatically
(`3.229.0-3.248.0`). HCL runtime support does not imply every target conversion
is valid: converting a Terraform program to the `hcl` target is rejected
(`3.249.0-3.254.0`).

## Resource and package forms

PCL supports parameterized providers and `read` blocks. A read locates a
resource by ID and queries it without registration. Engine snippets are PCL
blocks retained in state to track ad-hoc resources (`3.229.0-3.248.0`).

Engine deployment options can target snippets by UUID through
`TargetSnippets` (`3.249.0-3.254.0`).

PCL can represent secret configuration values and resource ranges; component
inputs are typechecked. Labels on `package` blocks are deprecated
(`3.214.1-3.228.0`).

## Types, defaults, and evaluation

Integer literals, list and tuple indexes, and the `element` and `range`
builtins use integer rather than generic number types. Maps of resources made
with `range` can be indexed by key (`3.229.0-3.248.0`).

PCL applies resource-schema defaults, resolves invoke-based configuration
defaults to the invoke result, and populates schema-declared nested output
fields so optional-object traversal is safe (`3.229.0-3.248.0`).

## Hooks and functions

PCL declares lifecycle hooks, including `onError`, and generators emit them to
Go, Node.js, and Python. A successful hook command retries the failed operation
(`3.229.0-3.248.0`, `3.249.0-3.254.0`).

Functions declared with multi-argument inputs are invoked positionally in PCL
(`3.249.0-3.254.0`). Go program generation can emit provider `Call` requests
(`3.214.1-3.228.0`).

## Imports and generated expressions

Import generation preserves assets, archives, resource references, and rich
values nested in maps and arrays. Map keys containing template sequences are
HCL-escaped (`3.229.0-3.248.0`). Converter and import workflows support
parameterized and extension-parameterized providers (`3.249.0-3.254.0`).

SDK and program generation support extension-parameterized packages, while
provider schemas support extension parameterization (`3.229.0-3.248.0`,
`3.249.0-3.254.0`).
