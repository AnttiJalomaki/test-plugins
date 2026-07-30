# State, imports, and backends

This reference includes guidance from `3.145.0-3.159.0`,
`3.160.0-3.181.0`, `3.182.0-3.198.0`, `3.199.0-3.214.0`,
`3.214.1-3.228.0`, `3.229.0-3.248.0`, and `3.249.0-3.254.0`.

## State repair and protection

### Delete state entries

`pulumi state delete --all` removes every resource entry from a stack in one
repair operation (`3.145.0-3.159.0`). The command also accepts multiple URNs
and dependency-orders their deletion (`3.214.1-3.228.0`). Export a recovery
copy first; state deletion does not delete the underlying cloud objects.

```shell
pulumi state delete '<URN-1>' '<URN-2>'
```

### Protect or force replacement

`pulumi state protect <URN>` sets protection directly in state
(`3.160.0-3.181.0`). `pulumi state taint` marks a resource for forced
replacement on the next update, while `pulumi state untaint` clears the marker
(`3.182.0-3.198.0`).

### Validate references

Unresolved references are deployment validation errors. Use
`--allow-dangling-references` only for a deliberate repair workflow
(`3.160.0-3.181.0`).

```shell
pulumi up --allow-dangling-references
```

### Checkpoint formats and backups

The CLI can read and write v4 checkpoints/deployments. On DIY backends,
`pulumi stack rm --remove-backups` removes stack backups as well
(`3.182.0-3.198.0`).

The builtin `Stash` resource can persist a value in state, and state accepts
floating-point `NaN` and infinity (`3.199.0-3.214.0`).

## Import workflows

### Adopt and update

A resource `import` option can adopt an existing resource and update it in the
same deployment. Its import ID is retained through updates, and generated .NET,
Go, Node.js, and Python programs emit the option (`3.160.0-3.181.0`). Resource
transforms may set the import option and the engine honors it
(`3.199.0-3.214.0`).

```typescript
const bucket = new aws.s3.Bucket("bucket", {}, { import: "existing-id" });
```

Importing with a noncanonical identifier no longer causes deletion on a later
update (`3.249.0-3.254.0`).

### Import files and rich values

Import files may define provider resources and associate imported resources
with them. Program generation handles asset and archive inputs, preserves
assets, archives, and resource references nested in maps and arrays, and
HCL-escapes map keys containing template sequences (`3.229.0-3.248.0`).

Resources in import files may provide inputs and outputs. When outputs are
present, Pulumi imports the supplied state directly and skips the provider
read. Parameterized and extension-parameterized providers are supported
(`3.249.0-3.254.0`).

Imports converted from `--from` state files always generate resources
(`3.182.0-3.198.0`). `pulumi import` supports parameterized packages and
providers (`3.145.0-3.159.0`).

### State-converter protocol

`ResourceImport` includes parent and properties fields for hierarchy and
property filtering. Converters can return provider resources, link imported
resources to them, receive a schema-loader target, and request mappings for a
named ecosystem rather than only their own (`3.249.0-3.254.0`).

### Automation API import

Generated Node.js, Python, and Go Automation APIs expose `import`. Go preview
with refresh and refresh operations can also pass `--import-pending-creates`
(`3.249.0-3.254.0`).

### Secrets managers during import

When imported state names the service-backed secrets manager, `pulumi stack
import` reconfigures it for the destination stack when needed
(`3.214.1-3.228.0`).

## DIY backends

### Stack tags

S3, Azure Blob, Google Cloud Storage, PostgreSQL, and local backends support
stack-tag CRUD, automatic system tags, and filtered stack listings. Tags are
versioned JSON in separate `.pulumi-tags` files beside checkpoints. Existing
untagged stacks remain compatible, and stack deletion removes the tag file
(`3.199.0-3.214.0`).

```shell
pulumi stack tag set environment production
pulumi stack ls --tag-filter environment=production
```

### Storage and project mode

DIY backends can zstd-compress state files. The legacy non-project operating
mode is deprecated (`3.214.1-3.228.0`).

### Journaling

Engine journaling is enabled by default. Set `PULUMI_DISABLE_JOURNALING=true`
only to opt out explicitly (`3.214.1-3.228.0`).

### Snapshot recovery

The service backend automatically repairs snapshot-integrity problems while
still emitting an error event for diagnosis (`3.214.1-3.228.0`).

## Backend migration

`pulumi stack migrate` moves a stack from another backend into the backend that
is currently logged in. It re-encrypts configuration secrets and stack state
under the destination backend's secrets provider (`3.249.0-3.254.0`). Verify
both backend identities and retain an export before migration.

## Stack-reference behavior

An undecryptable `StackReference` output is omitted (`3.160.0-3.181.0`). In
Python, looking up a nonexistent stack-reference output no longer raises an
exception (`3.199.0-3.214.0`).

Reading non-secret stack outputs and running `pulumi about` no longer requires
the passphrase for a passphrase-encrypted stack (`3.249.0-3.254.0`).
