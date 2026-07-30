# Testing and Go tooling

## Recover failed test cleanup

Since 1.7.0, when `tofu test` cannot clean up resources it dumps the state
file. Preserve that file and use it to recover and manage the remaining
objects.

## Mock providers and targeted overrides

OpenTofu 1.8.0 adds `mock_provider` with `mock_resource` and `mock_data`, plus
`override_resource`, `override_data`, and `override_module` for replacing
specific results:

```hcl
mock_provider "aws" {
  mock_resource "aws_instance" {
    defaults = {
      id = "i-test"
    }
  }
}

override_resource {
  target = aws_instance.web
  values = {
    public_ip = "192.0.2.10"
  }
}
```

Test-file `variables` blocks can reference variables. Run names cannot contain
spaces.

Later 1.8 patches stop validating mock-provider definitions against the real
provider schema and relax type checking for mocks and overrides. Do not rely
on that leniency indefinitely.

OpenTofu 1.9.0 allows `override_resource` and `override_data` inside a
particular `mock_provider`, limiting those overrides to the mock. Invalid mock
and override fields become errors rather than warnings.

## Remote modules and run outputs

OpenTofu 1.10.0 lets an explicit module under test in `.tftest.hcl` use a
remote source. Test-file `provider` blocks can refer to output values from a
`run` block.

## Iterated mocks and functional variables

OpenTofu 1.11.0 lets `mock_provider` use `for_each`, and test-file `variable`
blocks can call functions.

Generated mocks follow provider schemas more closely. Update mocks and
overrides that depended on formerly unchecked, invalid shapes.

## Go integration libraries

Introduced alongside 1.8.0, TofuDL is a Go library for:

- locating the latest OpenTofu release
- verifying its signature
- downloading and extracting the binary
- mirroring releases for air-gapped environments

The experimental `libregistry` library provides structured access to registry
metadata and primitives for independent registry tooling. Treat its API as
unstable while it remains experimental.
