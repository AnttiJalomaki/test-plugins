# Repository writes and authentication

Use this reference for commit safety, raw repository downloads, credential
selection, token locations, logout, and incident response.

## Optimistic concurrency for mutations

Repository mutation methods that accept `parent_commit` can treat it as the
expected branch head. Supply the commit against which the change was prepared.
If the branch moves, the mutation fails instead of silently applying on an
unexpected base.

This guard matters for overwrites, generated metadata, coordinated multi-file
changes, or any workflow in which a stale write could lose another writer's
update.

A safe retry loop should:

1. Read the current branch head and relevant files.
2. Compute the intended mutation against that state.
3. Submit the mutation with that head as `parent_commit`.
4. On a conflict, fetch the new state and recompute; do not blindly replay an
   overwrite.

`parent_commit` provides an expected-base check. It does not by itself turn a
sequence of separate mutations into one atomic transaction.

## Redirected repository downloads

A repository `resolve` URL can redirect to content-addressed storage. The
redirect target may be on a different origin.

Raw HTTP code must:

- follow the redirect when appropriate;
- compare the original and destination origins;
- avoid forwarding the bearer token to an unrelated origin;
- retain normal integrity and error checks on the downloaded content.

Prefer `hf_hub_download` or a supported client flow, which handles the
authentication boundary safely. Do not implement a redirect handler that
copies the original `Authorization` header unconditionally.

## Explicit token resolution

`HF_TOKEN` takes precedence over the token stored on disk.

For APIs that accept `token=`:

| Value | Meaning |
| --- | --- |
| credential string | Use that credential |
| `True` | Require or request the locally resolved token |
| `False` | Suppress authentication |

Examples:

```python
api.model_info("open/model", token=False)
api.model_info("org/private-model", token=True)
```

Use `token=False` when a read must remain anonymous even if the machine has a
saved credential. Set `HF_HUB_DISABLE_IMPLICIT_TOKEN=1` to keep an available
token off otherwise-anonymous reads throughout an environment.

Explicitly passing a string is useful for isolated jobs, but avoid exposing it
through logs, process arguments, exception text, or committed configuration.

## Token storage location

`HF_TOKEN_PATH` overrides the path of the stored-token file under `HF_HOME`.
Cache variables do not move this file:

- `HF_HUB_CACHE` controls Hub cache placement;
- `HF_XET_CACHE` controls Xet cache placement;
- neither one relocates authentication state.

When relocating all application state, configure credential and cache paths
separately. Review their ownership and permissions independently as well.

## Logout is not remote revocation

`logout()` and `hf auth logout` remove saved credentials from the local
environment. They do not revoke the remote token.

For a compromised or retired token:

1. Remove local saved credentials.
2. Revoke the token in Hub settings.
3. Replace it in every remaining deployment that should retain access.
4. Review logs and repository activity as appropriate.

Changing a local token path or deleting a cached login record is not a
substitute for remote revocation.

## Downloaded private content survives logout

Logging out does not delete files already downloaded from private
repositories. Hub snapshots and Xet chunks are storage state, not credential
state.

On a decommissioned or shared machine, handle all three boundaries:

- remove or rotate local credentials;
- revoke remote credentials when required;
- inspect and deliberately remove private cached bytes according to retention
  policy.

Use the supported cache commands for Hub cache content and account separately
for the Xet chunk cache. See
[files-cache-uploads.md](files-cache-uploads.md) for cache-layer behavior.

## Authentication review checklist

- Decide whether each operation should be authenticated, locally resolved, or
  explicitly anonymous.
- Treat `HF_TOKEN` as higher priority than a stored token.
- Configure `HF_TOKEN_PATH` independently from cache locations.
- Use supported download helpers for cross-origin content redirects.
- Never forward a bearer token blindly across origins.
- Pair local logout with remote revocation when retiring a token.
- Treat cached private bytes as a separate cleanup responsibility.
