# State and plan encryption

This reference covers the encryption configuration introduced in
`1.7-state-encryption` and later changes that affect migration and input
handling.

## Build the encryption graph

OpenTofu can encrypt local or backend state and saved plans independently. A
configuration declares a key provider, feeds its result to an encryption
method, and assigns that method to `state`, `plan`, or both:

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

`TF_ENCRYPTION` contains the body of an `encryption` block and is merged over
the configuration in code. `enforced = true` prevents plaintext output when,
for example, an environment-supplied encryption method is absent.

Encryption variables and locals must normally resolve during `tofu init`.
They cannot depend on state data or provider-defined functions.

## Migrate plaintext and roll keys

Enabling encryption does not make existing plaintext readable. Make the new
method primary and explicitly allow the old representation as a fallback:

```hcl
method "unencrypted" "migration" {}

state {
  method = method.aes_gcm.main
  fallback {
    method = method.unencrypted.migration
  }
}
```

Reads try the primary method before fallbacks; every write uses the primary
method. After a successful `tofu apply` rewrites the state, remove the
plaintext fallback. The same sequence rolls encryption keys or methods.

To decrypt deliberately, reverse the relationship: make `unencrypted` the
primary method, retain the old encrypted method as a fallback, disable
enforcement, apply, and only then remove the encryption configuration.

OpenTofu 1.9.0 automatically applies migration when the encryption
configuration changes, but the primary/fallback model still determines how
artifacts are read and rewritten.

## Keep metadata names stable

Encrypted artifacts contain metadata tied to key-provider and method names.
Renaming either can make stored data unreadable. Roll to renamed methods with
a fallback. When producer and consumer configurations need different names,
assign a stable `encrypted_metadata_alias` before the names diverge.

Documented key providers and methods are guaranteed for only one additional
minor release. `tofu plan` and `tofu apply` warn when a component is
deprecated; migrate it before the following minor upgrade.

## Decrypt remote state separately

The current project's state encryption does not configure decryption for
`terraform_remote_state`. Configure remote-state consumers independently. A
default can apply to all data sources, while named entries override it.
Selectors can be `<name>`, `<module>.<name>`, or indexed forms such as
`<module>.<name>[0]`.

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

## Choose providers and AES-GCM keys

The PBKDF2 key provider accepts a passphrase of at least 16 characters or the
result of another provider. Its defaults are:

- 32-byte derived key
- 600,000 iterations
- 32-byte salt
- SHA-512, with SHA-256 also supported

Cloud-backed providers are:

- `aws_kms`: `kms_key_id`, `key_spec`, and S3-style authentication
- `gcp_kms`: `kms_encryption_key`, `key_length`, and GCS-style authentication
- `azure_vault`: `vault_uri`, `vault_key_name`, `key_length`, always with
  Entra ID
- `openbao`: `key_name`, optional `BAO_TOKEN` and `BAO_ADDR`, and an optional
  Transit-engine path

```hcl
method "aes_gcm" "main" {
  keys = key_provider.aws_kms.main
}
```

AES-GCM requires a 16-, 24-, or 32-byte provider key. Prefer a derivation
provider or a key-management system with rotation over a short static key;
repeated AES-GCM key use eventually encounters key-saturation limits.

## External key providers and methods

Experimental external hooks can retrieve keys or perform encryption. An
external key provider runs one `command`; an external method has distinct
`encrypt_command` and `decrypt_command` arrays and may consume a provider
result:

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

Programs first emit one protocol header:

```json
{"magic":"OpenTofu-External-Key-Provider","version":1}
```

```json
{"magic":"OpenTofu-External-Encryption-Method","version":1}
```

Key-provider input is JSON `null` during encryption or stored metadata during
decryption. Its response contains base64 encryption and decryption keys plus
optional metadata. External-method input and output contain a base64
`payload` and an optional base64 `key`.

## Apply-time inputs and backend encryption

OpenTofu 1.11.0 permits input values during apply for state and plan
encryption. Every non-ephemeral input must still equal its planned value.
From 1.11.4, JSON-form method configuration accepts `keys` as a normal
expression or a template interpolation rather than requiring interpolation.

OpenTofu 1.12.0 expands backend-side encryption options: AzureRM supports
Customer-Provided Keys and Customer-Managed Keys for server-side encryption.
This protects backend storage and is distinct from OpenTofu's artifact
encryption graph.

Early 1.12 releases also had a security defect involving OpenBao-wrapped
state-encryption data. Use 1.12.4 or later.
