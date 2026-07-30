---
name: opentofu-knowledge-patch
description: OpenTofu
version: 1.12.0
license: MIT
metadata:
  author: Nevaberry
---


# OpenTofu Knowledge Patch

Use this skill when writing, reviewing, upgrading, testing, or operating
OpenTofu configurations. Check the project's required OpenTofu version and
lock file first. Apply only guidance available to that version, and prefer the
configuration, tests, provider schemas, and observed behavior when they differ
from general guidance.

## Reference index

| Reference | Topics |
|---|---|
| [state-and-plan-encryption.md](references/state-and-plan-encryption.md) | Encryption graphs, migrations, rollover, remote state, key providers, external hooks |
| [language-modules-and-lifecycle.md](references/language-modules-and-lifecycle.md) | Early evaluation, provider functions, expressions, ephemeral values, moves, removals, lifecycle |
| [backends-distribution-and-platforms.md](references/backends-distribution-and-platforms.md) | Backend changes, credentials, locking, registries, caches, integrity, operating-system boundaries |
| [cli-automation-and-output.md](references/cli-automation-and-output.md) | Planning selectors, output modes, JSON, state inspection, console, diagnostics, tracing |
| [testing-and-go-tooling.md](references/testing-and-go-tooling.md) | Test mocks and overrides, cleanup, remote modules, test variables, TofuDL, libregistry |

## Upgrade blockers and deprecations

### Remove obsolete S3 compatibility settings

- OpenTofu 1.8 removes the S3 backend's `use_legacy_workflow` argument. Remove
  it before upgrading; standard AWS CLI and SDK credential precedence is the
  only workflow.
- S3 module sources adopt that same credential discovery in 1.11, so an
  upgrade can select a different credential source.
- OpenTofu 1.12 removes `OPENTOFU_USER_AGENT`; do not use it to replace the
  default HTTP User-Agent.

### Do not mix PostgreSQL locking generations

OpenTofu 1.10 changes PostgreSQL backend locking. Never run 1.10 and older
processes against the same database: their incompatible locks can allow
conflicting writes and data loss.

### Reconfigure AzureRM authentication

OpenTofu 1.11 ignores `endpoint`/`ARM_ENDPOINT` and
`msi_endpoint`/`ARM_MSI_ENDPOINT`. Use `MSI_ENDPOINT` where applicable, avoid
combining `environment` with `metadata_host`, and refresh the working directory:

```bash
tofu init -reconfigure
```

Use `-reconfigure`, not `-migrate-state`, because these authentication changes
do not move state.

### Respect security and platform floors

- OpenTofu 1.10 requires Linux kernel 3.2+ or macOS 11+.
- OpenTofu 1.11 requires macOS 12+, rejects SHA-1 TLS signatures, and rejects
  malformed SSH certificates signed by a certificate key.
- Use 1.11.4+ when installing providers or modules from potentially untrusted
  ZIP archives.
- Use the latest 1.12 patch available, at least 1.12.4 for the documented
  security fixes and safe saved plans involving `lifecycle.destroy = false`.
- OpenTofu 1.12 is the last planned macOS 12 series. WinRM warns in 1.12 and is
  planned to fail in 1.13; move Windows provisioners to SSH.
- The `ghcr.io/opentofu/opentofu` image is not supported as a base for custom
  images from 1.10 onward.

## Encrypt state and saved plans safely

State and saved-plan encryption are configured independently. Connect a key
provider to a method, then assign the method to `state`, `plan`, or both:

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
  }
}
```

Existing plaintext or old-key artifacts need an explicit fallback. Reads try
the primary method and then fallbacks; writes always use the primary. Run a
successful operation to rewrite the artifact before removing the fallback.

Encryption values normally must be available during initialization. OpenTofu
1.11 can also accept apply-time inputs, but every non-ephemeral value must
match the planned value. Configure `terraform_remote_state` decryption
separately from encryption of the current project's state.

See [state-and-plan-encryption.md](references/state-and-plan-encryption.md) for
metadata-name stability, supported key providers, AES-GCM requirements,
external protocol shapes, decryption, and remote-state selectors.

## Treat initialization as a separate evaluation phase

OpenTofu can evaluate variables and locals early for backend arguments and
module `source` and `version`. These expressions must be available before
providers and state. Sensitive values cannot be used where initialization or
module installation would expose them.

```hcl
variable "module_source" {
  type  = string
  const = true
}

module "network" {
  source  = var.module_source
  version = var.module_version
}
```

Use `const = true` in OpenTofu 1.12 to state the static-evaluation contract.
For older 1.8 configurations, `TOFU_ENABLE_STATIC_SENSITIVE=1` opts into the
sensitive marking that became standard in 1.9.

An identically named `.tofu` file masks its `.tf` counterpart. This lets a
module keep a Terraform-compatible fallback while placing OpenTofu-only syntax
in the `.tofu` file.

## Use current lifecycle and state-refactoring primitives

OpenTofu 1.10 supports provider-assisted cross-resource-type `moved` blocks
and richer `removed` blocks with lifecycle and provisioner configuration.
OpenTofu 1.12 adds dynamic `prevent_destroy`, provider-defined import
identities, and state-only removal directly on a managed resource:

```hcl
resource "example_object" "detached" {
  lifecycle {
    prevent_destroy = var.protect
    destroy         = false
  }
}
```

`destroy = false` forgets the object instead of asking the provider to destroy
it. Use 1.12.4+ when a saved plan might replace such a resource.

For optional single-instance resources and modules, OpenTofu 1.11 provides
`lifecycle.enabled`. From 1.11.4, modules with local provider configurations
reject it, just as they reject `count`, `for_each`, and `depends_on`.

## Keep secrets out of plans and state

OpenTofu 1.11 adds ephemeral variables, outputs, and provider-defined
resources. Their values live in memory for one phase and are never persisted
to a plan or state. Provider-defined write-only managed-resource attributes
likewise accept secret input without retaining it. Both features require
provider schema support.

Do not infer that an ordinary sensitive value is ephemeral: sensitivity hides
display, while ephemerality controls persistence.

## Use planning and automation controls deliberately

`-exclude` removes an address and its dependants from a plan; `-target`
includes an address and its requirements. OpenTofu 1.10 also accepts reusable
address lists:

```bash
tofu plan -target-file=targets.txt
tofu plan -exclude-file=deferred.txt
tofu apply -concise
```

Use `-show-sensitive` only when disclosure is intentional. For simultaneous
human and machine output in OpenTofu 1.12, write JSON events separately:

```bash
tofu plan -json-into=plan-events.json
```

The target can be a regular file or, for streaming consumers, an IPC object
such as a named pipe or `/dev/fd/N`.

## Inspect configuration, state, and plans explicitly

Prefer the explicit OpenTofu 1.10 selectors:

```bash
tofu show -state
tofu show -plan=PLANFILE
```

OpenTofu 1.11 can emit configuration JSON without creating a plan:

```bash
tofu show -json -config
tofu show -json -config -module=modules/example
```

The configuration form includes variable type constraints and whether each
variable is required. See
[cli-automation-and-output.md](references/cli-automation-and-output.md) for
diagnostic consolidation, console input and locking, concise modes, JSON
schemas, destroy controls, and experimental initialization tracing.

## Test with scoped doubles

Use `mock_provider` for provider-wide fake schemas and data, and
`override_resource`, `override_data`, or `override_module` for specific
targets. Overrides may be nested in a mock provider when they should apply
only to that mock.

Mocks deliberately became stricter: invalid mock or override fields are
errors, and newer generated values follow provider schemas more closely.
Correct stale test shapes instead of depending on unchecked fields.

When cleanup fails, `tofu test` writes state so remaining objects can be
recovered. Test-file providers can use prior run outputs, mocked providers can
use `for_each`, and test variables can call functions. Full examples and
version-specific constraints are in
[testing-and-go-tooling.md](references/testing-and-go-tooling.md).
