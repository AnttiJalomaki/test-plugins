# CLI, automation, distribution, and libraries

This reference covers command and integration surfaces from `1.7.0` through
`1.12.0`.

## Concise and structured output

`tofu plan -concise` omits state-refresh logs since 1.7. `tofu init -json` and
`tofu get -json` expose JSON output for automation.

Since 1.10, `tofu apply -concise` suppresses progress-like output while
retaining final results for non-streaming automation.

In 1.12, `-json-into=FILENAME` writes the same machine-readable stream as
`-json` while preserving normal human output on stdout:

```bash
tofu plan -json-into=plan-events.json
```

Streaming commands can target an IPC object such as a named pipe or
`/dev/fd/N` for concurrent consumption.

## Generated and schema output

Since 1.8, `tofu plan -generate-config-out=generated.tf` renders JSON-shaped
values with `jsonencode(...)`, rather than quoting JSON text.

`tofu providers schema -json` includes provider-defined function schemas.

Since 1.11, configuration can be inspected without first producing a plan:

```bash
tofu show -json -config
tofu show -json -config -module=modules/example
```

The configuration JSON includes each input variable's type constraint and
whether the input is required.

## Plan selection

Since 1.9, `tofu plan -exclude=ADDRESS` skips the selected object and every
dependent object. This complements `-target`, which includes selected objects
and their requirements.

Since 1.10, reusable address lists can be loaded from files:

```text
tofu plan -target-file=targets.txt
tofu plan -exclude-file=deferred.txt
```

## Sensitive and diagnostic output

Since 1.9, `-show-sensitive` unmasks sensitive values for `tofu plan`,
`tofu apply`, and other commands that return configuration or state. Use it
only when the output channel is trusted.

Commands also accept `-consolidate-warnings` and `-consolidate-errors` to
control diagnostic summarization.

`tofu console` accepts multiline expressions within brackets or across
backslash-escaped newlines. In 1.12 it also accepts `-lock=false` and
`-lock-timeout=DURATION`:

```bash
tofu console -lock-timeout=30s
```

## Explicit state and plan inspection

OpenTofu 1.10 adds explicit forms:

```text
tofu show -state
tofu show -plan=PLANFILE
```

They select current state or a saved plan. The older positional form remains
supported.

## Destroy behavior

In 1.12, this command suppresses errors for objects forgotten during destroy
and exits successfully:

```bash
tofu destroy -suppress-forget-errors
```

Use it only when forgetting those objects is intentional.

## Validation and registry client controls

Since 1.11, `tofu validate` can validate a non-root module that declares extra
provider configurations through `configuration_aliases`.

Registry retry counts and request timeouts can be configured in the CLI
configuration as well as through environment variables.

## XDG and browser environment behavior

OpenTofu follows XDG Base Directory locations since 1.7.

`OPENTOFU_USER_AGENT`, which fully replaced the default HTTP User-Agent, is
removed in 1.12. On Unix, `tofu login` honors `BROWSER` when it contains a
single command that accepts the URL as its sole argument. An inherited
`BROWSER` value can therefore change login's launch behavior.

## Experimental initialization tracing

Since 1.10, environment-controlled OpenTelemetry tracing can send partial
`tofu init` traces to a collector controlled by the operator. The feature is
experimental and currently exposes limited detail.

## Go integration libraries

TofuDL locates the latest OpenTofu release, verifies its signature, downloads
it, and extracts the binary. It also provides tooling to mirror releases into
air-gapped environments.

The experimental `libregistry` library provides structured registry metadata
and building blocks for independent registry tooling. Treat its API as
unstable.
