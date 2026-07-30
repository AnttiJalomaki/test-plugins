# CLI and operations

Source batches represented here include `3.145.0-3.159.0`,
`release-notes-117`, `3.160.0-3.181.0`, `3.182.0-3.198.0`,
`3.199.0-3.214.0`, `3.214.1-3.228.0`, `3.229.0-3.248.0`, and
`3.249.0-3.254.0`.

## Engine operation selection and execution

### Exclude targets

Target-aware operations accept `--exclude <URN>`; add `--exclude-dependents`
to omit children of the excluded target too (`3.145.0-3.159.0`).

```shell
pulumi up --exclude '<URN>' --exclude-dependents
```

### Run the program before refresh or destroy

Starting with CLI 3.160.0, `refresh` and `destroy` accept `--run-program`. Use it when changed program
code creates short-lived credentials, fetches secrets, defines dynamic
providers, or supplies other operation context. Without it, these commands use
updated stack configuration but do not run changed program code
(`release-notes-117`).

`PULUMI_RUN_PROGRAM` supplies the global setting. `preview` and `up` accept
`--run-program` when combined with `--refresh` (`3.160.0-3.181.0`).

```shell
PULUMI_RUN_PROGRAM=true pulumi up --refresh
pulumi preview --refresh --run-program
```

`refresh` and `destroy` also accept `--config` and `--config-path`
(`3.199.0-3.214.0`). Inline Automation API equivalents are covered in the
automation reference.

### Override environment and validation per operation

`up`, `preview`, `destroy`, and `refresh` accept `--override-env` to substitute
imported environments for one run without editing stack configuration. They
also accept `--skip-config-validation` to bypass the project's config schema
for that run (`3.249.0-3.254.0`).

### Parallelism

`PULUMI_PARALLEL` supplies `--parallel`; `PULUMI_PARALLEL_DIFF` enables
concurrent diff calculations. Effective parallelism observes container cgroup
limits (`3.145.0-3.159.0`).

```shell
PULUMI_PARALLEL=8 PULUMI_PARALLEL_DIFF=true pulumi up
```

### Confirmations and plugin installation

`PULUMI_SKIP_CONFIRMATIONS` is honored anywhere the CLI asks for confirmation.
`--skip-plugin-pre-install` bypasses up-front plugin installation
(`3.229.0-3.248.0`). These controls do not make a destructive operation safe;
validate backend, stack, and targets independently.

## Configuration and environment

### Select a stack without persistent selection

`PULUMI_STACK` selects the stack for configuration and state commands
(`3.145.0-3.159.0`).

```shell
PULUMI_STACK=dev pulumi config get region
```

### Supply any flag with `PULUMI_OPTION_*`

Every command flag has a generic environment form such as
`PULUMI_OPTION_REFRESH` for `--refresh`. `pulumi about` reports `PULUMI_*`
variables, and `pulumi about env` helps inspect the environment
(`3.199.0-3.214.0`). Template sources can be overridden with:

- `PULUMI_TEMPLATE_GIT_REPOSITORY`
- `PULUMI_TEMPLATE_BRANCH`
- `PULUMI_POLICY_TEMPLATE_GIT_REPOSITORY`
- `PULUMI_POLICY_TEMPLATE_BRANCH`

### Set explicit configuration types

`pulumi config set --type` selects the stored scalar type
(`3.145.0-3.159.0`). `pulumi config set-all --json` accepts bulk JSON input,
and Automation API provides `SetAllConfigJson` (`3.199.0-3.214.0`).

```shell
pulumi config set featureEnabled true --type bool
```

`pulumi config set --raw` preserves newlines read from stdin
(`3.229.0-3.248.0`). Pulumi YAML warns when YAML `null` would become an empty
string. The temporary 3.170.0 typed-YAML preservation behavior was reverted in
3.174.0 and must not be assumed (`3.160.0-3.181.0`). Pulumi YAML does accept
`object` as a configuration type and parses such values as objects
(`3.182.0-3.198.0`).

Refreshing stack configuration includes the environments used by that stack
(`3.214.1-3.228.0`).

### Show secrets only in trusted output

`preview` and `up` accept `--show-secrets`, which writes plaintext secrets to
both the terminal and captured logs (`3.145.0-3.159.0`).

## Output and diagnostics

### Structured summaries

`up`, `preview`, `refresh`, `destroy`, and `import` accept `--output json`. For
the first four, the summary includes affected resources with URN, type, name,
operation, and parent (`3.229.0-3.248.0`).

```shell
pulumi preview --output json
```

`--output <format>` also works on stack list/history/tag list, policy
list/group list, project list, config environment list, and plugin list
(`3.249.0-3.254.0`).

### Human and streaming output

CLI diagnostics go to stderr, `--show-full-output` defaults to false, and
`PULUMI_ENABLE_STREAMING_JSON_PREVIEW` controls streaming JSON previews
(`3.182.0-3.198.0`). `--urns` shows full URNs in preview, up, destroy, refresh,
import, and watch displays (`3.229.0-3.248.0`). Strings containing non-UTF-8
bytes render as `b"<base64>"` in diffs and JSON output
(`3.249.0-3.254.0`).

## Logging and observability controls

Automatic encrypted CLI log capture is enabled for every command. The earlier
`PULUMI_ENABLE_AUTOMATIC_LOGGING` opt-in from `3.229.0-3.248.0` is superseded.
Logs live under `~/.pulumi/logs`, property-value secrets are redacted, and
`pulumi logs decrypt`, `ls`, `rm`, and `share` manage them
(`3.249.0-3.254.0`).

`TRACEPARENT` parents CLI spans beneath an existing trace
(`3.229.0-3.248.0`). For OTLP details, see the automation and integrations
reference.

## Project and template commands

Projects may omit a runtime in CLI and Automation API settings. `pulumi project
new -y` writes a minimal project without a template, `pulumi new` aliases
`pulumi project new`, and the `pulumi` package supports `npx pulumi`
(`3.229.0-3.248.0`).

`pulumi new` accepts qualified Registry template names and lists templates
published through `pulumi template publish`. Private Registry templates no
longer require `PULUMI_EXPERIMENTAL`; set
`PULUMI_DISABLE_REGISTRY_RESOLVE=true` to disable Registry resolution
(`3.182.0-3.198.0`). Pulumi Cloud templates are also valid `pulumi new`
sources (`3.145.0-3.159.0`).

The `pulumi query` command was removed in 3.157.0 and has no
replacement-compatible CLI surface; scripts must stop invoking it
(`3.145.0-3.159.0`).

## Direct Cloud and resource commands

### `pulumi api`

`pulumi api <operation-or-path>` calls Pulumi Cloud operations or paths with
field, header, input, body, path-template, content-negotiation, and dry-run
handling. `list` and `describe` show the OpenAPI surface. `--paginate` combines
cursor pages into one JSON envelope, `--emit-events` writes pagination progress
to stderr, and the final formatting flag is `--output`, not `--format`
(`3.229.0-3.248.0`).

### `pulumi do`

The initial direct resource `create`, `patch`, and `delete` operations required
`--stateless`; `--provider` could take configuration from provider state, and
project-scoped PCL inputs received the selected stack's organization and short
name (`3.229.0-3.248.0`).

Stateful `create` and `delete` plus a stateful `upsert` are now available.
Number and boolean inputs may be expressions (`3.249.0-3.254.0`). Retain
`--stateless` only when direct-provider behavior is intended.

## Neo workflows

`pulumi neo` is generally visible and runs requested shell and filesystem tools
locally in the working directory while using Pulumi Console chat. It supports
non-interactive `--print`, approval and permission modes,
`--disable-integrations`, and a plan mode selected before the first message.
Plan mode blocks writes, `pulumi up`, and PR creation until approval
(`3.229.0-3.248.0`).

`pulumi neo acp` exposes Neo as an Agent Client Protocol process over stdio,
with read-only and plan session modes. `pulumi neo resume` restores chat
history; `--debug-update` and `--debug-preview` investigate failed operations
(`3.249.0-3.254.0`).

## Deployment settings removal

The experimental local `Pulumi.<stack>.deploy.yaml` workflow is removed,
including `deployment settings init`, `pull`, `configure`, `env`, and
`push`/`update`/`up`, `--config-file`, and SDK file readers. Use
`pulumi deployment settings get`, `edit`, and `destroy` for Pulumi Cloud
settings (`3.249.0-3.254.0`).
