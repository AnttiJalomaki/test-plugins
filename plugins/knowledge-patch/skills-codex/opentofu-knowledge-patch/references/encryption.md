# State and plan encryption

This reference incorporates the `1.7-state-encryption`, `1.9.0`, `1.11.0`,
and `1.12.0` batch guidance.

## Configuration graph and evaluation

State and saved plans are encrypted independently. An encryption configuration
declares a key provider, passes its result to a method, and assigns that method
to `state`, `plan`, or both. `TF_ENCRYPTION` accepts the contents of the
`encryption` block and merges over configuration in code.

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

`enforced = true` prevents plaintext output when, for example, an
environment-supplied method is missing. Variables and locals used here must be
resolvable during `tofu init`; they cannot depend on state data or
provider-defined functions.

Since 1.11, inputs may also be supplied during apply for state and plan
encryption. Every non-ephemeral input must equal its planned value. From
1.11.4, JSON-form method configuration accepts `keys` as a normal expression
or a template interpolation.

## Plaintext migration, rollover, and decryption

Enabling encryption does not implicitly authorize reading an existing
plaintext artifact. Make the new method primary and explicitly permit the old
representation as a fallback:

```hcl
method "unencrypted" "migration" {}

state {
  method = method.aes_gcm.main
  fallback {
    method = method.unencrypted.migration
  }
}
```

Reads try the primary and then the fallback. Every write uses the primary, so a
successful `tofu apply` rewrites state. Only then remove the fallback.
Encryption-configuration changes apply migrations automatically since 1.9.

Use the same procedure for key or method rollover. To decrypt deliberately,
make `unencrypted` primary, place the old encrypted method in `fallback`,
disable enforcement, apply, and only then remove encryption configuration.
Back up state, plans, and keys before any migration.

## Stored metadata and compatibility

Encrypted artifacts store metadata tied to key-provider and method names.
Renaming either may make the artifact unreadable. Roll through a fallback, or
give the key provider a stable `encrypted_metadata_alias` before names need to
differ. The alias is also useful when producer and remote-state consumer
configurations use different labels.

Documented providers and methods are guaranteed for only one additional minor
release. `tofu plan` and `tofu apply` warn when an encryption component is
deprecated; migrate it before the following minor upgrade.

## Remote-state consumers

Decrypting `terraform_remote_state` is separate from encrypting the current
project. A default can cover every remote-state data source, and named entries
can override it. Labels may target `<name>`, `<module>.<name>`, or indexed
forms such as `<module>.<name>[0]`.

```hcl
remote_state_data_sources {
  default {
    method = method.aes_gcm.shared
  }
  remote_state_data_source "database.primary[0]" {
    method = method.aes_gcm.database
  }
}
```

## Key providers

PBKDF2 accepts a passphrase of at least 16 characters or a chained provider
result. Defaults are a 32-byte key, 600,000 iterations, a 32-byte salt, and
SHA-512; SHA-256 is also supported.

Cloud-backed providers and their primary inputs are:

- `aws_kms`: `kms_key_id`, `key_spec`, and S3-style authentication.
- `gcp_kms`: `kms_encryption_key`, `key_length`, and GCS-style authentication.
- `azure_vault`: `vault_uri`, `vault_key_name`, `key_length`, always using
  Entra ID.
- `openbao`: `key_name`, optional `BAO_TOKEN` and `BAO_ADDR`, and an optional
  Transit-engine path.

```hcl
method "aes_gcm" "main" {
  keys = key_provider.aws_kms.main
}
```

AES-GCM requires a 16-, 24-, or 32-byte provider key. Prefer a derivation
provider or rotating key-management system over a short static key; repeated
AES-GCM key use eventually reaches key-saturation limits.

## Experimental external hooks

An external key provider runs one `command`. An external method has separate
`encrypt_command` and `decrypt_command` arrays and may receive a key-provider
result.

```hcl
key_provider "external" "keys" {
  command = ["./keys"]
}
method "external" "cipher" {
  encrypt_command = ["./cipher", "--encrypt"]
  decrypt_command = ["./cipher", "--decrypt"]
  keys            = key_provider.external.keys
}
```

The program first emits one of these handshake objects:

```json
{"magic":"OpenTofu-External-Key-Provider","version":1}
{"magic":"OpenTofu-External-Encryption-Method","version":1}
```

Key-provider input is `null` during encryption and stored metadata during
decryption. It returns base64 encryption and decryption keys plus optional
metadata. Method input and output carry a base64 `payload` and an optional
base64 `key`. Treat both hook types as experimental.

## Backend-side encryption

OpenTofu configuration encryption protects state and plan content before
storage. Backend-side encryption is configured independently. The AzureRM
backend adds support in 1.12.0 for both Customer-Provided Keys and
Customer-Managed Keys for server-side encryption.
