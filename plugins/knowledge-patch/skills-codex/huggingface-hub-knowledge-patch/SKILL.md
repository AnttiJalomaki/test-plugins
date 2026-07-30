---
name: huggingface-hub-knowledge-patch
description: Hugging Face Hub
version: null
license: MIT
metadata:
  author: Nevaberry
---


# Hugging Face Hub

Use this skill when writing or reviewing code that uses `huggingface_hub`,
the `hf` command, Hub repository APIs, Hub caches, Xet-backed storage,
Inference Providers, Inference Endpoints, or Spaces.

Prefer the installed package's metadata and runtime behavior when they differ
from general guidance. Hub services and provider routes are remote state:
inspect their current state before making assumptions about availability,
credentials, persistence, or completion.

## Reference index

| Reference | Topics |
| --- | --- |
| [references/client-migration.md](references/client-migration.md) | Python and HTTPX compatibility, removed APIs and arguments, CLI migration, upload returns, Xet, packaging |
| [references/repositories-auth.md](references/repositories-auth.md) | Optimistic concurrency, redirected downloads, token resolution, token storage and revocation |
| [references/storage-transfers.md](references/storage-transfers.md) | Cache safety, cache cleanup, resumable uploads, futures, server-side copy, Xet/LFS interoperability |
| [references/inference-spaces.md](references/inference-spaces.md) | Routed inference, credentials, endpoints, Space metadata, persistence, secrets, OAuth |

## Quick migration checklist

### Replace removed client surfaces

- Replace `Repository` with `HfApi` commit-oriented operations or supported
  Git/Xet tooling. Account for different conflict, atomicity, and local
  worktree semantics.
- Replace `HfFolder` with `login`, `logout`, `auth_switch`, and `get_token`.
- Replace `InferenceApi` with `InferenceClient` or
  `AsyncInferenceClient`.
- Replace `huggingface-cli` automation with `hf`, including `hf auth`,
  `hf download`, `hf upload`, and `hf cache`.
- Replace `use_auth_token=` with `token=`.
- Remove `resume_download`, `force_filename`, and
  `local_dir_use_symlinks` from download calls.

### Migrate networking to HTTPX

The client uses HTTPX instead of `requests` and `aiohttp`.

- Catch transport failures through the `httpx.HTTPError` hierarchy.
- Catch Hub response failures through `HfHubHttpError`.
- Replace `configure_http_backend` with `set_client_factory` or
  `set_async_client_factory`.
- Configure proxies, TLS, timeouts, transports, and mocks on a global or
  custom HTTPX client, not with per-call `proxies=`.

Check the installed release's package metadata for its Python floor. The
initial v1 floor does not guarantee that every later 1.x release supports the
same Python versions.

### Update transfer assumptions

- Treat upload results as the documented commit-oriented result or URL type;
  do not rely on old file-CDN truthiness, string concatenation, or Git-wrapper
  behavior.
- Xet through `hf_xet` is the supported large-file path.
- Remove `hf_transfer` assumptions and
  `HF_HUB_ENABLE_HF_TRANSFER`.
- Enable `HF_XET_HIGH_PERFORMANCE` only after accepting its documented
  resource tradeoff.
- Keep framework serialization and model-card callbacks in their owning
  integration; removed TensorFlow/Keras helpers are not core-client APIs.
- Select optional extras or direct dependencies from the installed release,
  because optional and CLI dependency groups changed.

## Authentication decisions

### Resolve tokens explicitly

Token resolution follows these rules:

1. `HF_TOKEN` overrides the token stored on disk.
2. A string passed as `token=` uses that credential.
3. `token=True` requests the locally resolved credential.
4. `token=False` suppresses authentication.
5. `HF_HUB_DISABLE_IMPLICIT_TOKEN=1` prevents an available token from being
   attached to otherwise-anonymous reads.

```python
api.model_info("open/model", token=False)
api.model_info("org/private-model", token=True)
```

### Separate storage, logout, and revocation

`HF_TOKEN_PATH` moves the stored-token file under `HF_HOME`.
Changing `HF_HUB_CACHE` or `HF_XET_CACHE` alone does not move authentication
state.

`logout()` and `hf auth logout` remove saved local credentials but do not
revoke the remote token. Revoke a compromised or retired token in Hub settings
as a separate action. Logging out also leaves downloaded private content on
disk.

### Handle redirects without leaking credentials

A repository `resolve` request may redirect to content-addressed storage.
Raw HTTP clients must follow that redirect without forwarding the bearer token
to an unrelated origin. Prefer `hf_hub_download` or the supported client flow,
which handles this boundary.

## Safe repository writes

When a mutation accepts `parent_commit`, pass the branch head that the caller
observed whenever an overwrite or lost update would be unsafe. If the branch
moves, the operation then fails rather than applying to an unexpected base.

Use commit-oriented APIs with explicit atomicity expectations:

- `create_commit` may combine supported add, delete, and
  `CommitOperationCopy` operations.
- Eligible copies happen server-side without re-uploading bytes.
- A copy still creates a repository commit and is constrained by supported
  source, destination, revision, and repository contexts.
- `upload_large_folder` may make multiple commits; it is resumable, not one
  all-or-nothing transaction.

For an all-or-nothing release, upload to a staging branch or repository,
validate it, and then promote it.

## Cache and local-directory safety

Treat paths returned from the central Hub cache as immutable. Blob content can
be linked into more than one revision snapshot, so editing a returned cache
path can corrupt shared content or invalidate multiple snapshots. Copy the
file into a working directory before editing it.

When using `local_dir`:

- selected files are materialized in that directory;
- resume metadata is written below `.cache/huggingface`;
- that metadata should be excluded from publication; and
- cross-project deduplication is lower than with the central cache.

Use `hf cache ls`, `hf cache rm`, and `hf cache prune` for Hub cache
inspection and cleanup. Do not delete cache internals while active work may be
using them.

The Hub snapshot cache and Xet chunk cache are separate:

- deleting snapshots need not delete Xet chunks;
- deleting chunks does not update repository refs; and
- logging out does not erase already downloaded private bytes.

## Long-running upload work

`upload_large_folder` records hashing, pre-upload, and commit progress under
the source folder's `.cache/huggingface`. To preserve resumability:

- rerun against the same source folder and repository;
- retain the progress metadata;
- do not modify files during the upload; and
- remember that partial progress can already exist as multiple commits.

`run_as_future=True` creates background work in the current process, not a
durable remote job. Futures retain per-client queue order, but the process must
remain alive and every future must be inspected for failure. Stop scheduled
commit helpers and verify their final commit before process exit.

## Large-file checkout checks

Xet-backed repositories provide a compatibility bridge for legacy Git LFS
clients, but Xet and LFS differ in storage and performance behavior. A generic
Git clone can succeed while leaving metadata or pointer files instead of the
large bytes. Verify filters and pointer materialization, or use a Hub download
API.

## Inference routing

`InferenceClient(..., provider="auto")` chooses an available provider for a
supported model and task under current routing rules. This does not create or
select a dedicated Inference Endpoint.

Use a named provider or a deployed endpoint URL when processor, region,
isolation, scaling, billing, or optional chat behavior must be controlled.
The common client surface does not make those properties identical between
routes.

Match credentials to the route:

- Hub-routed inference can use a Hugging Face token with the required
  inference permissions and billing association.
- A direct partner-provider route uses that provider's documented key.
- Never send a partner credential to an arbitrary model repository URL.

## Endpoint lifecycle

Endpoint creation and updates are asynchronous. Poll the returned remote state
and handle terminal failure before sending production traffic.

- `scale_to_zero` retains configuration and permits a later request to
  cold-start serving.
- `pause` requires an explicit resume.
- Endpoint exposure is configured separately from the privacy of the source
  model repository.

## Space configuration and persistence

README fields `suggested_hardware` and `suggested_storage` are recommendations
for duplication or configuration. They do not allocate resources; use runtime
settings for actual hardware and persistent storage.

`preload_from_hub` can stage narrowly selected, revision-pinned Hub files
during build or startup. It does not replace dependency declarations. Custom
HTTP headers are restricted to the documented allowlist.

The ordinary Space filesystem is ephemeral across restarts and rebuilds.
Store durable state on separately provisioned persistent storage at its
documented mount, or in an external service.

- A sleeping Space wakes on access.
- A paused Space requires explicit restart or resume.
- Restarting does not promise preservation of ephemeral files.

Space variables are visible to users with settings access. Secrets become
write-only through the settings UI or API after creation; both are normally
injected into the runtime as environment variables.

README `hf_oauth` settings can provision OAuth callbacks, client settings, and
requested scopes for user login. They do not automatically authorize the
Space's server process to read private repositories.
