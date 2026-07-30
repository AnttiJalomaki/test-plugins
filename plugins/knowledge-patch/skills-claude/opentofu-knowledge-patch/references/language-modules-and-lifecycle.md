# Language, modules, and lifecycle

## Provider functions and templates

Since 1.7.0, providers can expose functions, including functions selected
dynamically from provider configuration. Invoke them with the provider
namespace:

```hcl
provider::<provider_name>::<funcname>(args)
```

`templatefile` can recursively call `templatefile`, with a default maximum
depth of 1024. This enables composition of file-backed templates without
external preprocessing.

## Early evaluation and OpenTofu-specific files

OpenTofu 1.8.0 can evaluate variables and locals early enough for backend
configuration and module `source` and `version`:

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

Only use values available during initialization; this release does not make
provider configuration dynamic. From 1.8.3,
`TOFU_ENABLE_STATIC_SENSITIVE=1` opts into sensitive marking for variables
used in backend configuration and module source/version. The 1.8 line warns
for compatibility, and marking becomes the default in 1.9.0.

OpenTofu 1.9 prompts for variables needed by early evaluation. It prohibits
sensitive values in backend configuration and module source locations because
initialization or installation would expose them.

OpenTofu 1.12.0 makes an initialization-time contract explicit:

```hcl
variable "module_source" {
  type  = string
  const = true
}
```

`const = true` requires every assigned value to be compatible with static
evaluation.

If `main.tofu` and `main.tf` both exist, OpenTofu ignores `main.tf`. A module
can therefore use `.tofu` for OpenTofu-only syntax and retain the identically
named `.tf` file as a Terraform-compatible fallback.

## Iterated provider configurations

OpenTofu 1.9.0 lets an aliased provider configuration use `for_each`.
`each.key` can configure each instance, and resources can select different
members:

```hcl
provider "aws" {
  alias    = "by_region"
  for_each = var.aws_regions
  region   = each.key
}
```

## Module contracts and selection expressions

Since 1.10.0, module authors can deprecate input variables and outputs;
consumers receive warnings. A module `version` may be `null`, which is
equivalent to omitting it.

Dynamic instance keys in a resource `provider` selection or module
`providers` mapping are automatically converted to strings. The built-in
`terraform` provider includes functions for encoding and decoding `.tfvars`
data and encoding arbitrary values as OpenTofu expression syntax.

OpenTofu 1.12 introduces a `language` configuration block that separates
OpenTofu constraints from constraints on other software. A module containing
this block requires OpenTofu 1.12+, so defer it when older OpenTofu versions
must remain supported.

## Expression behavior

OpenTofu 1.10.0 makes `&&` and `||` short-circuit. A skipped right operand
therefore cannot fail by dereferencing a null value. `element` extends its
wrapping behavior to negative indexes, where `-1` selects the final item:

```hcl
locals {
  enabled = var.settings != null && var.settings.enabled
  last    = element(var.items, -1)
}
```

In 1.11.0:

- `issensitive(unknown)` returns unknown. Do not feed that result to `count`,
  `for_each`, or another plan-time-only context unless the argument is known.
- Object constructors assigned to an object-typed input warn about undeclared
  attribute names.
- `regex` and `regexall` accept long Unicode properties such as `\p{Letter}`.
- `fileset` accepts backslash escapes for literal metacharacters.

In 1.12.0:

- Comparing a complex value with `null` produces a sensitive boolean only
  when the whole value is sensitive, not merely a nested attribute. The
  comparison can therefore drive plan-time `enabled` in more cases.
- `yamldecode` supports YAML `<<` merges with a sequence of mappings as well
  as a single mapping.

## Ephemeral and write-only data

OpenTofu 1.11.0 adds ephemeral input variables, output values, and
provider-defined resources. Their values exist only in memory for one
operation phase and never enter plans or state. Providers can also define
write-only managed-resource attributes so initial passwords, private keys,
and similar values can be submitted without retention.

Both ephemeral resources and write-only attributes require explicit provider
support. A sensitive value only suppresses display; it is not automatically
ephemeral.

## Conditional resources and modules

Use `lifecycle.enabled` from 1.11.0 for a resource or module that should have
zero or one instances:

```hcl
module "servers" {
  source  = "./app-cluster"
  servers = 5

  lifecycle {
    enabled = var.enable_cluster
  }
}
```

It is nested under `lifecycle`, avoiding collision with resource arguments or
module inputs named `enabled`. From 1.11.4, a module containing local provider
configurations rejects `enabled`, matching its restrictions on `count`,
`for_each`, and `depends_on`.

## Moves, removals, imports, and destruction

OpenTofu 1.10.0 lets `moved` blocks migrate remote objects between resource
instances of different types, with the provider handling state conversion.
`removed` blocks can include `lifecycle` and `provisioner` configuration to
control remaining instances.

OpenTofu 1.12.0 permits `lifecycle.prevent_destroy` to reference symbols in
the same module. It also allows a managed resource to set
`lifecycle.destroy = false`, removing the object from state without asking
the provider to destroy it:

```hcl
variable "protect_database" {
  type    = bool
  default = true
}

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

Use 1.12.4+ when saving a plan that might replace a resource with
`destroy = false`; earlier 1.12 releases fail in that case.

An `import` block can use a provider-defined `identity` object matching the
resource type's identity schema instead of a plain `id`.

`replace_triggered_by` now replaces a resource when the referenced resource
is itself being replaced; previously it reacted only when that reference was
updated.

## Deprecation diagnostics

OpenTofu 1.12 warns when configuration refers to an attribute or block marked
deprecated in provider schema. Use the `-deprecation=` CLI option when those
warnings must be disabled.
