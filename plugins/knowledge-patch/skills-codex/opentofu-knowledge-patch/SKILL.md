---
name: opentofu-knowledge-patch
description: OpenTofu
version: 1.12.0
license: MIT
metadata:
  author: Nevaberry
---


# OpenTofu Knowledge Patch

Use this skill when writing, reviewing, upgrading, or automating OpenTofu
configuration. Check the project's required version and lock file first, then
apply only behavior available to that version.

## Reference index

| Reference | Topics |
|---|---|
| [encryption.md](references/encryption.md) | State and plan encryption, migration, key providers, remote-state decryption |
| [language-modules-and-lifecycle.md](references/language-modules-and-lifecycle.md) | Early evaluation, `.tofu` files, expressions, providers, modules, ephemeral values, lifecycle |
| [state-backends-and-installation.md](references/state-backends-and-installation.md) | State persistence, S3, AzureRM, HTTP, PostgreSQL, locking, caches, package installation |
| [cli-automation-and-libraries.md](references/cli-automation-and-libraries.md) | CLI flags, JSON streams, inspection, registry distribution, tracing, Go libraries |
| [testing.md](references/testing.md) | Mocks, overrides, test variables, remote test modules, cleanup |
| [upgrade-security-and-platforms.md](references/upgrade-security-and-platforms.md) | Breaking upgrades, operating-system floors, transport hardening, deprecations |

## Upgrade blockers and deprecations

### Remove the legacy S3 credential switch

The S3 backend's standard AWS CLI/SDK credential workflow became the default
in 1.7. The compatibility argument was removed in 1.8:

```hcl
terraform {
  backend "s3" {
    # Remove before upgrading to 1.8:
    # use_legacy_workflow = true
  }
}
```

Review the selected credential after the upgrade. S3 module sources moved to
the same standard discovery behavior in 1.11.

### Do not mix PostgreSQL locking generations

The 1.10 `pg` backend uses finer-grained locks and a state layout that is not
safe to share with older processes. Mixing them against one database can allow
conflicting writes and data loss. Upgrade all writers together.

### Refresh AzureRM backend configuration

In 1.11, AzureRM ignores `endpoint`/`ARM_ENDPOINT` and
`msi_endpoint`/`ARM_MSI_ENDPOINT`. Replace the latter with `MSI_ENDPOINT`,
avoid setting `environment` together with `metadata_host`, then run:

```bash
tofu init -reconfigure
```

Do not use `-migrate-state` for this authentication/configuration refresh.

### Observe patch-level safety fixes

- Use 1.10.2+ when native S3 lockfiles require server-side encryption.
- Use 1.10.5+ when multiple processes share `TF_PLUGIN_CACHE_DIR`.
- Use 1.11.4+ when installation can encounter an untrusted ZIP archive.
- Use 1.12.4+ throughout the 1.12 line. Earlier patches have security defects
  involving SSH, OpenBao-wrapped encryption data, revoked SSH CA keys, and
  malicious Git URLs; they also fail when saving some replacement plans for
  resources with `lifecycle.destroy = false`.

### Check platform and transport boundaries

- 1.10 requires Linux kernel 3.2+ or macOS 11+.
- 1.11 requires macOS 12+, rejects SHA-1 TLS signatures, and rejects malformed
  SSH certificates signed by a certificate key.
- WinRM warns in 1.12 and is planned to become an error in 1.13; use SSH for
  Windows provisioners.
- The 1.12 series is the last planned macOS 12 line.

See [upgrade-security-and-platforms.md](references/upgrade-security-and-platforms.md)
for container-image, Windows-junction, and architecture details.

## Encrypting state and plans

Encryption connects a key provider to a method, then assigns the method
independently to state and saved plans:

```hcl
terraform {
  encryption {
    key_provider "pbkdf2" "main" {
      passphrase = var.state_passphrase
    }
    method "aes_gcm" "main" {
      keys = key_provider.pbkdf2.main
    }
    state {
      method   = method.aes_gcm.main
      enforced = true
    }
    plan {
      method   = method.aes_gcm.main
      enforced = true
    }
  }
}
```

`TF_ENCRYPTION` supplies and overrides the contents of the `encryption` block.
Encryption inputs must be available during initialization; they cannot depend
on state or provider-defined functions.

Never enable encryption over existing plaintext without a migration fallback.
Reads try the primary and then fallbacks, while writes always use the primary:

```hcl
method "unencrypted" "migration" {}

state {
  method = method.aes_gcm.main
  fallback {
    method = method.unencrypted.migration
  }
}
```

Apply successfully before removing the fallback. The same pattern handles key
rollover and deliberate decryption. Back up both artifacts and keys first.
Configure `terraform_remote_state` decryption separately. See
[encryption.md](references/encryption.md) for metadata aliases, provider
parameters, and external hook protocols.

## OpenTofu-specific module initialization

Variables and locals may drive module `source`, module `version`, and backend
arguments when their values are available during initialization:

```hcl
variable "module_source" {
  type  = string
  const = true
}

module "network" {
  source = var.module_source
}
```

`const = true` makes this static-input contract explicit. Sensitive values are
not allowed in backend configuration or module source locations.

An identically named `.tofu` file masks its `.tf` counterpart. This lets a
module keep a portable `.tf` fallback and place OpenTofu-only syntax in the
matching `.tofu` file.

## Ephemeral values and conditional instances

Ephemeral variables, outputs, and provider-defined resources live only in
memory for one operation phase and are not persisted in plans or state.
Provider-defined write-only resource attributes similarly accept secrets
without retaining them. Both features require provider support.

Use `lifecycle.enabled` for a resource or module that should have either zero
or one instance:

```hcl
module "servers" {
  source = "./app-cluster"

  lifecycle {
    enabled = var.enable_cluster
  }
}
```

A module with local provider configurations cannot use `enabled` from 1.11.4,
matching the restrictions on `count`, `for_each`, and `depends_on`.

## Destruction and refactoring

Protection and forget behavior can be selected dynamically:

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

`destroy = false` removes the object from state without asking the provider to
destroy it. Cross-type `moved` blocks can migrate state between resource types,
and `removed` blocks can contain lifecycle and provisioner configuration.
Provider-defined import identities allow structured `identity` objects in
`import` blocks.

## Automation highlights

- `-exclude` omits an object and its dependents; `-target` includes an object
  and its requirements. Reusable selectors live in `-exclude-file` and
  `-target-file`.
- `-json-into=FILENAME` keeps human output on stdout while writing the JSON
  event stream to a file or IPC endpoint.
- `tofu show -json -config` inspects configuration without creating a plan;
  use `-module=DIR` for one module.
- `tofu apply -concise` suppresses progress-like output but retains final
  results.
- `tofu destroy -suppress-forget-errors` succeeds despite errors for objects
  already forgotten during destroy.

Consult the references before relying on experimental encryption hooks or
initialization tracing, and pin patch releases where a documented safety fix
applies.
