# State, import, and resource lifecycle

This reference is organized by lifecycle task. Source batch attribution:
`3.145.0-3.159.0`, `3.160.0-3.181.0`, `3.182.0-3.198.0`,
`replace-with`, `3.199.0-3.214.0`, `3.214.1-3.228.0`,
`3.229.0-3.248.0`, and `3.249.0-3.254.0`.

## Target or exclude resources

Target-aware engine operations accept `--exclude <URN>`.
`--exclude-dependents` also omits children of the excluded resource.

```shell
pulumi up \
  --exclude 'urn:pulumi:dev::app::pkg:index:Type::name' \
  --exclude-dependents
```

Engine deployment options can also target state snippets by UUID through
`TargetSnippets`.

## Protect, retain, and delete

An explicit `protect: false` or `retainOnDelete: false` on a child overrides an
inherited true setting. A protected-delete failure no longer prevents unrelated
deployment work from continuing.

```typescript
const child = new aws.s3.Bucket("child", {}, {
  parent,
  protect: false,
  retainOnDelete: false,
});
```

`pulumi state protect <URN>` changes protection directly in state.
`pulumi state delete --all` deletes every resource entry. Newer CLIs accept
multiple URNs in one command and dependency-order their deletion.

```shell
pulumi state protect '<resource-URN>'
pulumi state delete '<URN-1>' '<URN-2>'
```

Back up state first. Deleting state does not delete the corresponding cloud
resource.

## Force and coordinate replacements

`pulumi state taint` marks a resource for forced replacement on the next
update; `pulumi state untaint` clears it.

From v3.207.0, `replaceWith` relates a custom resource to other resources.
Replacement of any referenced resource replaces the resource carrying the
option. Relationships can be transitive or mutual, allowing a whole group to
be replaced. The initial language support is Go, Python, Node.js, and Java;
C# and YAML were not yet supported in that release.

```typescript
const app = new aws.ec2.Instance("app-service", args, {
  replaceWith: [database],
});
```

`replacement_trigger` forces replacement when an arbitrary stored value
changes. Engine and Go support began in 3.208.0, followed by Node.js and Python.
Replacement triggers are passed through remote component `Construct` calls.

## Use lifecycle hooks and retries

The engine and Go, Node.js, and Python SDKs support resource hooks. Hooks pass
through component `Construct`, transforms can set them, and hook arguments
include resource type and name. Destroy operations with delete hooks must run
the program; after-delete hooks also run for component resources.

`OnError` hooks implement retry policies. All provider errors are forwarded to
error hooks. PCL can bind `onError`, and generated Go, Node.js, and Python
programs retry a failed operation when the hook command succeeds. PCL and
Python decorator forms can declare resource and error hooks; hook calls receive
resource options. A failed after-hook fails the deployment.

Node.js and Python hooks receive secret values as `Output` objects, preserving
secrecy rather than exposing an internal representation.

## Validate references and ignored changes

Unresolved resource references are validation errors unless the operation uses
`--allow-dangling-references`.

```shell
pulumi up --allow-dangling-references
```

When an `ignoreChanges` path is absent from old state, the engine now uses the
new value instead of raising an error. Hidden-diff resource options are exposed
as Go `HideDiffs`, Node.js `hideDiffs`, and Python `hide_diffs`.

Failed registrations now produce faulted outputs rather than unknown outputs,
and diffs nested within `Output` values are no longer skipped.

## Import and update resources

A resource with the `import` option can be adopted and updated in the same
deployment. Its import ID remains through updates, and .NET, Go, Node.js, and
Python program generation emits the import option.

```typescript
const bucket = new aws.s3.Bucket("bucket", {}, {
  import: "existing-id",
});
```

The engine honors import options introduced by resource transforms. Importing
under an ID different from the provider's canonical ID no longer causes a
later update to delete the resource.

Import files can:

- declare provider resources and associate imported resources with them;
- contain assets, archives, and resource references nested in maps and arrays;
- include resource inputs and outputs;
- bypass provider `Read` when outputs directly provide imported state;
- represent resources under parameterized and extension-parameterized
  providers.

Import code generation preserves nested assets and archives and HCL-escapes
map keys containing template sequences. Imports converted from `--from` state
files always generate resources.

The generated Node.js, Python, and Go Automation APIs expose `import`. Go
preview-refresh and refresh options can pass `--import-pending-creates`.

## Understand state formats and special values

The CLI reads and writes v4 checkpoints and deployments. Resource state can
contain non-finite floating values including `NaN` and infinity.

The builtin `Stash` resource stores an arbitrary value in state. Engine
snippets retain PCL blocks in state to track ad-hoc resources.

`StackReference` outputs that cannot be decrypted are elided. Looking up a
missing stack-reference output in Python returns the missing result rather than
raising.

When an invoke has secret inputs but its provider does not support secrets,
the engine marks invoke outputs secret. The general secrets filter no longer
treats case-insensitive `true` and `false` literals as filter values.

## Refresh provider and dynamic-resource state

Providers can request recovery refreshes for affected resources after a
partial failure; the engine honors the request by default.

Node.js and Python dynamic providers can return inputs from `read()` so a
refresh preserves the inputs needed for later diffs. The service backend
automatically repairs snapshot-integrity defects while emitting an error event
for diagnosis.

When imported state references a service-backed secrets manager, `pulumi stack
import` reconfigures it for the target stack as necessary.

## Migrate or remove backend state

`pulumi stack migrate` moves a stack from another backend into the logged-in
backend. It re-encrypts configuration secrets and state with the target
backend's secrets provider.

For DIY backends, `pulumi stack rm --remove-backups` removes stack backups as
well as the stack. Use this only with a separately verified backup.
