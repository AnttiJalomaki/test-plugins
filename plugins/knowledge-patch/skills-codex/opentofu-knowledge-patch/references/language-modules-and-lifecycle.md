# Language, modules, providers, and lifecycle

This reference organizes language guidance from `1.7.0`, `1.8.0`, `1.9.0`,
`1.10.0`, `1.11.0`, and `1.12.0`.

## Provider functions and templates

Providers can expose configuration-dependent functions. Call one with the
provider namespace:

```hcl
provider::<provider_name>::<funcname>(args)
```

The provider may select the implementation dynamically from its configuration.
Provider-defined function schemas also appear in
`tofu providers schema -json`.

Since 1.7, `templatefile` may call `templatefile` recursively. The default
maximum call depth is 1024, enabling composition of file-backed templates
without a separate preprocessor.

## Early-evaluated module and backend inputs

Since 1.8, variables and locals may be evaluated early enough for module
`source`, module `version`, and backend arguments. Expressions must use values
available during initialization; dynamic provider configuration was not part
of the 1.8 feature.

```hcl
variable "module_version" {
  default = "5.1.0"
}

locals {
  state_key = "production.tfstate"
}

terraform {
  backend "s3" {
    key = local.state_key
  }
}

module "network" {
  source  = "example/network/aws"
  version = var.module_version
}
```

OpenTofu prompts for variables required by early evaluation since 1.9.
Sensitive values are prohibited in backend configuration and module source
locations because initialization or module installation would expose them.

On the 1.8 line from 1.8.3, set
`TOFU_ENABLE_STATIC_SENSITIVE=1` to opt into sensitive marking for variables
used in module sources, module versions, and backend configuration. Without
the opt-in, 1.8 warns for compatibility; the behavior is the default in 1.9.

Since 1.12, `const = true` explicitly requires an input to be compatible with
static evaluation:

```hcl
variable "module_source" {
  type  = string
  const = true
}
```

## OpenTofu-specific files and constraints

If both `main.tofu` and `main.tf` exist, OpenTofu ignores `main.tf`. A module
can therefore retain a Terraform-compatible `.tf` fallback while using
OpenTofu-only syntax in the identically named `.tofu` file.

The 1.12 `language` configuration block separates OpenTofu version constraints
from constraints for other software. A module containing this block requires
OpenTofu 1.12 or later, so do not adopt it when older-version compatibility is
required.

## Provider configuration instances

Since 1.9, an aliased provider configuration may use `for_each`. `each.key`
configures each instance, and different resource instances can select
different provider instances.

```hcl
provider "aws" {
  alias    = "by_region"
  for_each = var.aws_regions

  region = each.key
}
```

Since 1.10, dynamic instance keys used in a resource's `provider` selection or
a module's `providers` map are converted to strings automatically.

## Module interface contracts

Module authors can mark input variables and output values deprecated since
1.10. Consumers referencing those interfaces receive warnings.

A module `version` expression may be `null` since 1.10, which is equivalent to
omitting the argument.

## Boolean, indexing, and encoding expressions

Since 1.10, `&&` and `||` short-circuit. A skipped right operand is not
evaluated and therefore cannot fail while dereferencing a null value.
`element` also wraps negative indexes, with `-1` selecting the last value.

```hcl
locals {
  enabled = var.settings != null && var.settings.enabled
  last    = element(var.items, -1)
}
```

The built-in `terraform` provider has functions for encoding and decoding
`.tfvars` data and for encoding arbitrary values as OpenTofu expression
syntax.

## Sensitivity and validation

Since 1.11, `issensitive` returns unknown when its argument is unknown.
Configurations that feed this result into plan-time-only contexts such as
`count` or `for_each` must first ensure the argument is known.

Other 1.11 expression changes are:

- Object constructors assigned to object-typed inputs warn about undeclared
  attribute names.
- `regex` and `regexall` accept long Unicode property names such as
  `\p{Letter}`.
- `fileset` accepts literal metacharacters escaped with backslashes.

Since 1.12, comparing a complex value with `null` produces a sensitive boolean
only when the whole value is sensitive, not merely when a nested attribute is
sensitive. The result can therefore feed plan-time contexts such as
`lifecycle.enabled`.

`yamldecode` now supports a YAML `<<` merge tag whose value is a sequence of
mappings, as well as a single mapping.

References to provider schema attributes or blocks marked deprecated produce
warnings. Use `-deprecation=` to control or disable those diagnostics.

## Ephemeral and write-only values

Since 1.11, input variables, output values, and provider-defined resources can
be ephemeral. Their values exist in memory for one operation phase and are
never persisted in saved plans or state.

Providers can expose write-only managed-resource attributes so an initial
password, private key, or similar secret can be submitted without OpenTofu
retaining it. Ephemeral resource types and write-only attributes both require
explicit provider support.

## Conditional resources and modules

`lifecycle.enabled` expresses the zero-or-one-instance case without using
`count = condition ? 1 : 0`:

```hcl
module "servers" {
  source  = "./app-cluster"
  servers = 5

  lifecycle {
    enabled = var.enable_cluster
  }
}
```

From 1.11.4, a module containing local provider configurations rejects
`enabled`, matching its existing restrictions on `count`, `for_each`, and
`depends_on`.

## Destruction controls

Since 1.12, `lifecycle.prevent_destroy` can refer to symbols in the same
module, including input variables. A managed resource can also set
`lifecycle.destroy = false` to remove the object only from state:

```hcl
resource "example_database" "main" {
  lifecycle {
    prevent_destroy = var.protect_database
  }
}

resource "example_object" "detached" {
  lifecycle {
    destroy = false
  }
}
```

Use 1.12.4 or later when saving a plan that may replace a resource with
`destroy = false`; earlier 1.12 releases fail in that case.

`replace_triggered_by` now triggers when the referenced resource is itself
being replaced. Previously it reacted only when the reference was updated.

## Moves, removals, and imports

Since 1.10, `moved` blocks may move remote objects between instances of
different resource types while migrating state. `removed` blocks may contain
`lifecycle` and `provisioner` configuration to control the remaining
instances.

Since 1.12, an `import` block may use an `identity` object conforming to the
provider-defined identity schema for the resource type instead of a plain
`id` string.
