# Resources and engine behavior

This reference consolidates resource behavior from `3.145.0-3.159.0`,
`release-notes-117`, `3.160.0-3.181.0`, `3.182.0-3.198.0`,
`replace-with`, `3.199.0-3.214.0`, `3.214.1-3.228.0`,
`3.229.0-3.248.0`, and `3.249.0-3.254.0`.

## Inheritance, protection, and retention

An explicit `false` on a child overrides inherited `protect: true` and
`retainOnDelete: true`. A protected-delete error no longer stops unrelated
deployment work (`3.145.0-3.159.0`).

```typescript
const child = new aws.s3.Bucket("child", {}, {
  parent,
  protect: false,
  retainOnDelete: false,
});
```

Autonaming configuration is stable as of 3.146.0 and applies to custom
resources, not component resources (`3.145.0-3.159.0`). It can disable generated names
globally or configure autonaming per component without program changes; the
feature is available from CLI 3.91.1 (`release-notes-117`).

Provider-option inheritance during registration no longer incorrectly carries
default providers across package boundaries (`3.214.1-3.228.0`).

## Replacement and diff controls

### `replaceWith`

A custom resource can use `replaceWith` to reference resources whose
replacement must also replace it. Relationships are transitive and may be
mutual to replace an entire group. Go, Python, Node.js, and Java support this;
C# and YAML support is still forthcoming (`replace-with`).

```typescript
const app = new aws.ec2.Instance("app-service", {}, {
  replaceWith: [database],
});
```

### Arbitrary triggers and hidden diffs

`replacement_trigger` forces replacement whenever its arbitrary value changes
between program runs. Engine and Go support arrived in 3.208.0, followed by Node.js
and Python (`3.199.0-3.214.0`). Remote-component `Construct` calls propagate
replacement triggers (`3.214.1-3.228.0`).

Go `HideDiffs`, Node.js `hideDiffs`, and Python `hide_diffs` hide selected
resource diffs (`3.199.0-3.214.0`). Diffs nested inside `Output` values are no
longer ignored (`3.249.0-3.254.0`).

### `ignoreChanges`

If an `ignoreChanges` path is absent from old state, the engine uses the new
value for that path rather than failing (`3.182.0-3.198.0`).

## Resource lifecycle hooks

Go, Node.js, and Python support resource hooks. Hooks pass through component
`Construct`; transforms can set them; and hook arguments include resource type
and name. Destroy involving delete hooks must run the program. After-delete
hooks run for component resources (`3.182.0-3.198.0`).

Node.js and Python hooks receive secrets as `Output` values rather than an
internal representation (`3.199.0-3.214.0`). The engine and these three SDKs
also support `OnError` hooks for custom retry policies
(`3.214.1-3.228.0`).

PCL can declare hooks, including `onError`; Python supplies decorator forms.
Hooks receive resource options, a failing after-hook fails the deployment, and
all provider errors reach error hooks. A successful error-hook command retries
the operation (`3.229.0-3.248.0`, `3.249.0-3.254.0`).

## Timeouts and failed values

`customTimeouts` has a `read` field for provider read timeouts
(`3.229.0-3.248.0`). Failed registrations now produce faulted outputs instead
of unknown outputs (`3.249.0-3.254.0`). Node.js and Python `Output.recover`
can catch and recover from errors raised during output resolution
(`3.229.0-3.248.0`).

## Providers and resource registration

Provider `Configure` receives the provider resource URN and ID. Explicit
providers receive `DiffConfig` calls for replacement decisions just like
default providers. Go `AnalyzerResourceOptions` exposes `Parent`, and resource
transforms receive the parent URN (`3.145.0-3.159.0`).

Node.js provider constructors receive `ignoreChanges`, `replaceOnChanges`,
`customTimeouts`, `retainOnDelete`, and `deletedWith` rather than dropping
them (`3.160.0-3.181.0`).

Provider resources accept `EnvVarMappings` to remap environment variables
before handing them to the provider (`3.214.1-3.228.0`).

Node.js and Python dynamic-provider `read()` methods may return inputs so a
refresh retains the inputs needed by future diffs (`3.214.1-3.228.0`).

## Invokes and secrets

Go, Node.js, and Python avoid invokes whose resource dependencies remain
unknown. Node.js and Python also wait for dependencies discovered in input
properties; plain invokes continue to accept output arguments for compatibility
(`3.145.0-3.159.0`).

When an invoke has a secret input and the provider lacks secret support, the
engine marks its outputs secret (`3.214.1-3.228.0`). The general CLI secrets
filter no longer treats case-insensitive `true` and `false` as filter values.

Provider invokes receive a `preview` flag (`3.199.0-3.214.0`). Calls honor a
provider reference in `__self__`, and the engine supplies `Name` and `Type` in
wire `ResourceReference` values (`3.214.1-3.228.0`).

## Views and recovery refresh

Engine views are enabled by default as of 3.176.0; Pulumi YAML enables them by
default as of 3.177.0 (`3.160.0-3.181.0`).

Providers can request default refresh of affected resources after partial
failures, and the engine honors that request as of 3.178.0
(`3.160.0-3.181.0`).

## Resource inspection

.NET, Go, Java, Node.js, Python, and YAML can locate the project root. Go and
Python expose `pulumiResourceName` and `pulumiResourceType` for a resource's
runtime name and type token (`3.145.0-3.159.0`).

Go and Python mock monitors can return registered resources for test
assertions. Go also exposes the current stack export map through
`GetCurrentExportMap` (`3.182.0-3.198.0`).
