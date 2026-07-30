# Policy, security, and authentication

This reference draws on `release-notes-117`, `native-oidc`,
`3.145.0-3.159.0`, `3.160.0-3.181.0`, `3.199.0-3.214.0`,
`3.214.1-3.228.0`, `3.229.0-3.248.0`, and `3.249.0-3.254.0`.

## OIDC and Pulumi Cloud login

`pulumi login` accepts an external identity provider's OIDC token and exchanges
it for a short-lived Pulumi Cloud token. The token lasts two hours by default;
`--oidc-expiration` changes the lifetime. `--oidc-token` accepts a raw token or
`file://` path. The Pulumi organization must trust the issuer and authorize the
token's claims and audience (`native-oidc`).

```shell
pulumi login \
  --oidc-token file:///var/run/secrets/token \
  --oidc-org my-org \
  --oidc-team platform-team
```

Use `--oidc-team` or `--oidc-user` to narrow the identity. When a token is
given, the CLI defaults to Pulumi Cloud and can infer organization, team, and
user from JWT claims (`3.214.1-3.228.0`).

## Authentication lifecycle

When `credentials.json` contains an OAuth refresh token, a 401 triggers one
automatic access-token refresh and retry (`3.229.0-3.248.0`).

Logout deletes all backend configuration, shared temporary agent credentials,
and the current tokenless backend. This is a breaking behavior change; inspect
the active login before automating logout (`3.229.0-3.248.0`).

CLI-launched providers receive the active login through `PULUMI_API` and
`PULUMI_ACCESS_TOKEN` (`3.229.0-3.248.0`).

## Secret propagation and display

`preview` and `up --show-secrets` place plaintext secret values in terminal and
captured output (`3.145.0-3.159.0`).

When an invoke has a secret input but its provider cannot accept secrets, the
engine marks outputs secret. The CLI's general secrets filter does not treat
case-insensitive `true` and `false` as filter values
(`3.214.1-3.228.0`). Node.js and Python hooks receive secrets as `Output`
values (`3.199.0-3.214.0`). PCL can declare configuration values that must be
read as secrets (`3.214.1-3.228.0`).

Automatic encrypted logs redact property-value secrets and can be shared with
`pulumi logs share` (`3.249.0-3.254.0`).

Undecryptable `StackReference` outputs are omitted
(`3.160.0-3.181.0`). Non-secret output reads and `pulumi about` no longer need
the passphrase for passphrase-encrypted stacks (`3.249.0-3.254.0`).

## ESC operations

ESC rotates long-lived credentials such as AWS IAM access keys on a schedule or
on demand. Rotation is declared in environments, keeps two secrets for smooth
consumer transitions, supports separate administrator and consumer roles, and
can invoke downstream-update webhooks (`release-notes-117`).

The Pulumi ESC GitHub Action injects environment secrets/configuration into a
workflow and can run ESC commands. Pair it with `pulumi/auth-actions` for
OIDC-based tokenless Pulumi Cloud authentication (`release-notes-117`).

`pulumi env open-request` submits a request for approval rather than leaving a
draft. Use `--reason` to add approver context (`3.249.0-3.254.0`).

## Policy-pack lifecycle

`pulumi policy install` installs policy-pack dependencies. Policy operations
automatically install missing policy analyzer plugins
(`3.214.1-3.228.0`).

The engine supports policy-severity overrides, and the CLI displays each
violation's severity (`3.199.0-3.214.0`). Go Automation API preview and update
options can carry policy packs (`3.160.0-3.181.0`).

## Analyze existing state and files

`pulumi policy analyze` evaluates a policy pack against existing stack state;
local packs can resolve ESC environments (`3.229.0-3.248.0`).

`pulumi policy analyze --file <stack-export>` evaluates an exported state file
without selecting a stack or logging into a backend. `pulumi policy new`
accepts `--runtime-options`, and policy list/group list accept structured
`--output` (`3.249.0-3.254.0`).

## Insights-discovered resources

Pulumi Insights can apply existing Policy as Code rules to discovered resources
that are not managed by Pulumi IaC. Link Insights Accounts to Policy Groups
alongside stacks to cover discovered AWS, Azure, OCI, and Kubernetes resources
(`release-notes-117`).
