# CLI operations and configuration

This reference is organized by operator task. Source batch attribution:
`3.145.0-3.159.0`, `release-notes-117`, `3.160.0-3.181.0`,
`3.182.0-3.198.0`, `3.199.0-3.214.0`, `3.214.1-3.228.0`,
`3.229.0-3.248.0`, and `3.249.0-3.254.0`.

## Configure execution

- `PULUMI_PARALLEL` supplies `--parallel`, and `PULUMI_PARALLEL_DIFF=true`
  enables concurrent diff calculation. Effective parallelism observes
  container cgroup limits.
- `PULUMI_STACK` selects the stack for config and state commands without a
  persistent `pulumi stack select`.
- Every command flag can use a `PULUMI_OPTION_*` environment variable. Convert
  a flag such as `--refresh` to `PULUMI_OPTION_REFRESH`.
- `PULUMI_SKIP_CONFIRMATIONS` is honored at every CLI confirmation prompt.
- `--skip-plugin-pre-install` bypasses up-front plugin installation.
- `up`, `preview`, `destroy`, and `refresh` accept `--override-env` for
  operation-scoped imported environments and `--skip-config-validation` for
  bypassing project config-schema checks.

```shell
PULUMI_PARALLEL=8 PULUMI_PARALLEL_DIFF=true pulumi up
PULUMI_STACK=dev pulumi config get region
PULUMI_OPTION_REFRESH=true pulumi preview
```

## Decide when to run the program

From CLI 3.160.0, `refresh` and `destroy` accept opt-in `--run-program`. Use it
when changed code establishes short-lived credentials, loads secrets, defines
dynamic providers, or supplies other operation context. Without it, these
commands use updated stack configuration but not changed program code.

`preview` and `up` also accept `--run-program` when combined with `--refresh`.
`PULUMI_RUN_PROGRAM` supplies the setting globally.

```shell
pulumi refresh --run-program
pulumi destroy --run-program
pulumi preview --refresh --run-program
PULUMI_RUN_PROGRAM=true pulumi up --refresh
```

Automation API inline programs have corresponding run-program controls for
refresh and destroy; see the language SDK reference.

## Handle config and environments

- `pulumi config set --type` selects the stored scalar type rather than
  relying on string handling.
- `pulumi config set --raw` preserves newlines from piped standard input.
- `pulumi config set-all --json` accepts bulk JSON input. Automation API calls
  the equivalent operation `SetAllConfigJson`.
- `refresh` and `destroy` accept both `--config` and `--config-path`.
- Refreshing stack configuration includes environments used by the stack.
- Pulumi YAML accepts `object` as a configuration type and parses its values
  as objects.
- YAML `null` produces a warning because it is read as an empty string. The
  3.170.0 change that preserved YAML types was reverted in 3.174.0; do not
  rely on that interim behavior.
- `pulumi config env list` supports `--output <format>`.

```shell
pulumi config set featureEnabled true --type bool
printf '%s' 'line one
line two' | pulumi config set message --raw
```

## Control display and machine output

`up`, `preview`, `refresh`, `destroy`, and `import` accept `--output json` for
a structured operation summary. For the first four operations, affected
resources include URN, type, name, operation, and parent.

```shell
pulumi preview --output json
```

Additional output contracts:

- CLI diagnostics go to stderr rather than stdout.
- `--show-full-output` defaults to false.
- `PULUMI_ENABLE_STREAMING_JSON_PREVIEW` controls streaming JSON previews.
- `--urns` displays full URNs in preview, up, destroy, refresh, import, and
  watch output.
- `--show-secrets` on `preview` and `up` writes plaintext secrets to the
  terminal and captured logs. Use only in controlled environments.
- Strings containing non-UTF-8 bytes render as `b"<base64>"` in diffs and JSON
  output.
- `pulumi stack list`, `stack history`, `stack tag list`, `policy list`,
  `policy group list`, `project list`, `config env list`, and `plugin list`
  accept `--output <format>`.

## Capture and export traces and logs

`--otel-traces` writes traces to a relative path or exports them to a gRPC
endpoint. Exporters accept `grpcs://`, authentication headers, and
`OTEL_RESOURCE_ATTRIBUTES`; provider OpenTracing spans are bridged into
OpenTelemetry. `TRACEPARENT` parents CLI spans beneath an existing trace.

```shell
pulumi preview --otel-traces ./traces.json
```

Automatic encrypted CLI logging was initially opt-in through
`PULUMI_ENABLE_AUTOMATIC_LOGGING`, but is now enabled for every command by
default. Logs are stored under `~/.pulumi/logs`, and property-value secrets are
redacted.

```shell
pulumi logs ls
pulumi logs decrypt
pulumi logs share
pulumi logs rm
```

## Authenticate and inspect the environment

`pulumi login` accepts an external OIDC token directly or from `file://`,
exchanges it for a short-lived Pulumi Cloud token, and avoids storing a
long-lived CI credential. The default lifetime is two hours and can be changed
with `--oidc-expiration`. The organization must trust the issuer and authorize
the claims and audience.

```shell
pulumi login \
  --oidc-token file:///var/run/secrets/eks.amazonaws.com/serviceaccount/token \
  --oidc-org my-org \
  --oidc-team platform-team
```

`--oidc-team` and `--oidc-user` narrow the resulting identity. Current login
defaults to Pulumi Cloud when given an OIDC token and can infer organization,
team, and user from JWT claims. This behavior originated in source batch
`native-oidc`.

When `credentials.json` includes an OAuth refresh token, a 401 triggers one
automatic access-token refresh and retry. `pulumi logout` now deletes all
backend configuration, shared temporary agent credentials, and the current
tokenless backend.

`pulumi about` reports `PULUMI_*` environment variables; `pulumi about env`
provides a focused environment view. Reading non-secret stack outputs and
running `pulumi about` no longer require the passphrase for a
passphrase-encrypted stack.

## Work with templates, projects, and local plugins

- `pulumi new` can consume Pulumi Cloud templates. Qualified registry template
  names are supported, and `PULUMI_DISABLE_REGISTRY_RESOLVE=true` disables
  registry resolution.
- Template sources can be overridden with
  `PULUMI_TEMPLATE_GIT_REPOSITORY`, `PULUMI_TEMPLATE_BRANCH`,
  `PULUMI_POLICY_TEMPLATE_GIT_REPOSITORY`, and
  `PULUMI_POLICY_TEMPLATE_BRANCH`.
- Projects can omit a runtime. `pulumi project new -y` writes a minimal
  project, and `pulumi new` is an alias of `pulumi project new`.
- Installing the `pulumi` package permits `npx pulumi`.
- `pulumi plugin run` starts a local binary plugin. Debug a source plugin with
  the exact form `--attach-debugger plugin=<name>`.
- Windows executable lookup considers `.cmd` and `.ps1`.
- The on-demand HCL runtime is not bundled; `pulumi convert --from hcl`
  installs its converter automatically.

`pulumi package publish` was introduced experimentally in 3.158.0 and
graduated in 3.166.0. `pulumi template publish`, added experimentally in CLI
3.180.0, remains experimental.

## Removed CLI surfaces

- `pulumi query` was removed in 3.157.0.
- The experimental local `Pulumi.<stack>.deploy.yaml` workflow was removed,
  including `deployment settings init`, `pull`, `configure`, `env`, and
  `push`/`update`/`up`; `--config-file` and matching SDK helpers are gone.
- Manage deployment settings with `pulumi deployment settings get`, `edit`,
  and `destroy` in Pulumi Cloud.
