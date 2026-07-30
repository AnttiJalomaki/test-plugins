# CLI, automation, and output

## Concise and JSON output

Since 1.7.0, `tofu plan -concise` omits state-refresh logs. `tofu init -json`
and `tofu get -json` provide machine-readable automation output.

OpenTofu 1.8.0 changes
`tofu plan -generate-config-out=generated.tf` to render JSON-shaped values
with `jsonencode(...)` instead of quoted JSON strings.

OpenTofu 1.10.0 adds `tofu apply -concise`, which suppresses progress-like
output while retaining final results for non-streaming automation.

OpenTofu 1.12.0 can preserve human output while writing the equivalent JSON
event stream to another target:

```bash
tofu plan -json-into=plan-events.json
```

Streaming commands may write to an IPC object such as a named pipe or
`/dev/fd/N` for concurrent consumption.

## Plan inclusion and exclusion

OpenTofu 1.9.0 adds transitive exclusion:

```bash
tofu plan -exclude=kubernetes_manifest.crds
```

`-exclude` skips the chosen object and everything depending on it. By
contrast, `-target` includes the chosen object and its requirements.

OpenTofu 1.10.0 adds reusable address-list files:

```bash
tofu plan -target-file=targets.txt
tofu plan -exclude-file=deferred.txt
```

## Sensitive and diagnostic output

OpenTofu 1.9.0 adds `-show-sensitive` to `plan`, `apply`, and other commands
that return configuration or state data. It deliberately unmasks sensitive
values, so restrict captured output and logs.

Commands also accept `-consolidate-warnings` and `-consolidate-errors` to
control diagnostic summarization.

OpenTofu 1.12.0 emits provider-schema deprecation warnings for referenced
attributes and blocks. Use `-deprecation=` to control those diagnostics.

## Console behavior and locking

Since 1.9.0, `tofu console` accepts multiline expressions within brackets or
across backslash-escaped newlines.

OpenTofu 1.12.0 adds console state-lock controls:

```bash
tofu console -lock=false
tofu console -lock-timeout=30s
```

## Inspect state and saved plans

OpenTofu 1.10.0 adds explicit forms:

```bash
tofu show -state
tofu show -plan=PLANFILE
```

These select current state or a saved plan. The older positional form remains
supported.

## Inspect configuration without a plan

OpenTofu 1.11.0 emits configuration JSON without first creating a plan:

```bash
tofu show -json -config
tofu show -json -config -module=modules/example
```

The module form inspects one directory. Configuration JSON includes every
input variable's type constraint and whether it is required.

`tofu validate` can also validate a non-root module that declares additional
provider configurations through `configuration_aliases`.

## Provider schemas and generated configuration

Since 1.8.0, `tofu providers schema -json` includes provider-defined function
schemas. Automation parsing generated configuration should also account for
the release's use of `jsonencode(...)` for JSON-shaped imported values.

## Force unlock and destroy controls

The HTTP backend supports `tofu force-unlock` from 1.10.0.

OpenTofu 1.12.0 adds:

```bash
tofu destroy -suppress-forget-errors
```

It suppresses errors for objects forgotten during destroy and exits
successfully.

## User-Agent compatibility

`OPENTOFU_USER_AGENT`, which replaced the default HTTP User-Agent completely,
is removed in 1.12.0. Remove automation that depends on that environment
variable.

## Experimental initialization tracing

OpenTofu 1.10.0 can emit partial OpenTelemetry traces for `tofu init` under
environment control. Send them only to a collector controlled by the
operator. The integration is experimental and its trace detail is limited.
