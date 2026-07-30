# OpenTofu test behavior

This reference covers test changes from `1.7.0`, `1.8.0`, `1.9.0`, `1.10.0`,
and `1.11.0`.

## Cleanup recovery

When `tofu test` cannot clean up created resources, it dumps the state file.
Use that state to recover and manage the resources that remain.

## Mock providers and defaults

Since 1.8, `mock_provider` can define `mock_resource` and `mock_data` defaults:

```hcl
mock_provider "aws" {
  mock_resource "aws_instance" {
    defaults = {
      id = "i-test"
    }
  }
}
```

Later 1.8 patch releases stop validating mock-provider definitions against the
real provider schema and relax type validation for mocks and overrides.

Generated mocks in 1.11 follow provider schemas more closely. Upgrade any mock
or override that depended on an invalid, previously unchecked shape.

## Targeted overrides

`override_resource`, `override_data`, and `override_module` replace a
particular result:

```hcl
override_resource {
  target = aws_instance.web
  values = {
    public_ip = "192.0.2.10"
  }
}
```

Since 1.9, `override_resource` and `override_data` may be nested inside a
specific `mock_provider`, scoping the override to that mock. Invalid mock and
override fields are errors rather than warnings.

## Variables and run outputs

Test-file `variables` blocks may reference variables since 1.8. Test run names
cannot contain spaces.

Since 1.10, test-file `provider` blocks may refer to output values from a
previous `run` block.

Since 1.11, test-file `variable` blocks may call functions.

## Iterated mocks

`mock_provider` supports `for_each` since 1.11, allowing a test to construct
multiple mock provider configurations from one declaration.

## Modules under test

Since 1.10, an explicit module under test in `.tftest.hcl` may use a remote
source rather than only a local source.
