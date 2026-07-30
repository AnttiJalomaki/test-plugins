---
name: huggingface-hub-knowledge-patch
description: Hugging Face Hub
version: null
license: MIT
metadata:
  author: Nevaberry
---


# Hugging Face Hub Knowledge Patch

Use this skill when working with `huggingface_hub`, repository files and
commits, authentication, caches, large-file storage, routed inference,
Inference Endpoints, or Spaces.

Prefer the installed package metadata, live API signatures, repository state,
and returned remote state over assumptions about a whole release family.

## Reference index

| Reference | Topics |
| --- | --- |
| [references/client-migration.md](references/client-migration.md) | Python and HTTP client compatibility, removed APIs and CLI, download arguments, upload results, Xet, framework integration |
| [references/repositories-auth.md](references/repositories-auth.md) | Commit concurrency, redirected downloads, token resolution and storage, logout and revocation |
| [references/files-cache-uploads.md](references/files-cache-uploads.md) | Cache immutability and cleanup, `local_dir`, resumable and deferred uploads, server-side copies, Xet/LFS interoperability |
| [references/inference-spaces.md](references/inference-spaces.md) | Routed providers, credentials, endpoint lifecycle, Space configuration, persistence, secrets, OAuth |

## Start with breaking changes

### Check the runtime floor

Do not assume that every release in a major series supports the same Python
minimum. Read the installed distribution metadata when choosing an interpreter
or declaring a downstream package constraint.

### Migrate transport customization to HTTPX

The synchronous and asynchronous clients use HTTPX rather than `requests` and
`aiohttp`.

- Catch `httpx.HTTPError` for transport failures.
- Catch the appropriate `HfHubHttpError` subclass for Hub response failures.
- Replace `configure_http_backend` with `set_client_factory` or
  `set_async_client_factory`.
- Put proxy, TLS, timeout, transport, and mock configuration on the global or
  custom HTTPX client.
- Do not pass the removed per-call `proxies=` argument.

### Replace removed abstractions

| Removed surface | Current direction |
| --- | --- |
| `Repository` | `HfApi` commit operations or supported Git/Xet tooling |
| `HfFolder` | `login`, `logout`, `auth_switch`, and `get_token` |
| `InferenceApi` | `InferenceClient` or `AsyncInferenceClient` |
| `huggingface-cli` | `hf` |

Commit-oriented HTTP operations are not a drop-in local Git worktree wrapper.
Re-evaluate conflict handling, atomicity, and assumptions about local files.

Current command families include:

```bash
hf auth
hf download
hf upload
hf cache
```

### Rename and remove call arguments

Use `token=` instead of the removed `use_auth_token` alias:

```python
api.model_info("org/model", token=token)
snapshot_download("org/model")
```

Download functions no longer accept `resume_download`, `force_filename`, or
`local_dir_use_symlinks`.

- Supported cache behavior handles resumption where applicable.
- Use the returned path instead of forcing an old filename convention.
- Use `force_download` only when a fresh fetch is required.
- Choose between the central cache and current `local_dir` behavior explicitly.

### Inspect upload return types

Upload methods return commit-oriented information or URLs, not the old
file-CDN abstraction. Follow the exact installed return type. Do not rely on an
old result's truthiness, string concatenation behavior, or local Git-wrapper
semantics.

### Remove legacy transfer switches

Xet through `hf_xet` is the supported large-file path and is integrated
automatically. The removed `hf_transfer` integration is not re-enabled by
`HF_HUB_ENABLE_HF_TRANSFER`.

Enable `HF_XET_HIGH_PERFORMANCE` only after accepting its documented resource
tradeoff.

### Move framework-specific behavior outward

Removed TensorFlow and Keras helpers should be replaced by framework-owned
serialization and model-card callbacks that call Hub primitives. Packaging
must select an extra or direct dependency that actually exists for the
installed release; optional and CLI dependency groups have changed.

## Repository and authentication quick reference

### Protect concurrent writes

For a mutation that accepts `parent_commit`, pass the branch head that the
change was prepared against. If the branch moved, the operation fails instead
of applying to an unexpected base.

Use this guard whenever a blind retry could overwrite another writer or
publish a change based on stale content.

### Resolve credentials deliberately

`HF_TOKEN` takes precedence over a token stored on disk.

For APIs accepting `token=`:

- a string uses that credential;
- `True` requests the locally resolved token;
- `False` suppresses authentication.

Set `HF_HUB_DISABLE_IMPLICIT_TOKEN=1` when otherwise-anonymous reads must not
silently use an available token.

```python
api.model_info("open/model", token=False)
api.model_info("org/private-model", token=True)
```

### Separate storage, logout, and revocation

`HF_TOKEN_PATH` moves the stored-token file under `HF_HOME`. Changing
`HF_HUB_CACHE` or `HF_XET_CACHE` does not move authentication state.

`logout()` and `hf auth logout` delete saved local credentials. They do not
revoke the remote token, and they do not erase private content already present
in caches.

### Handle download redirects safely

Repository `resolve` requests can redirect to content-addressed storage. A raw
HTTP client must follow redirects without forwarding a bearer token to an
unrelated origin. Prefer `hf_hub_download` or another supported client flow,
which handles this boundary.

## Files, cache, and upload quick reference

### Never edit central-cache results

The central cache stores content in `blobs` and exposes
`snapshots/{commit}` trees through links where supported. A path returned from
that cache can be shared by several snapshots.

Copy cached content to a working directory before modification.

With `local_dir`, selected files are materialized in the destination and
transfer metadata is written under `.cache/huggingface`. Exclude that metadata
from publication and expect less cross-project deduplication.

### Use cache commands

Use `hf cache ls`, `hf cache rm`, and `hf cache prune`. Avoid deleting cache
internals while work is active.

The Hub cache and Xet chunk cache are separate layers. Removing repository
snapshots may leave chunks, while clearing chunks does not update repository
refs.

### Treat large-folder upload as resumable, not atomic

`upload_large_folder` records hashing, pre-upload, and commit progress in the
source folder's `.cache/huggingface`.

- Keep the metadata for a retry against the same folder and repository.
- Do not modify source files during the run.
- Expect multiple commits.
- For all-or-nothing publication, upload to a staging branch or repository,
  validate it, then promote it.

### Await process-local background work

`run_as_future=True` returns a future and preserves per-client queue order, but
the work is tied to the current process. Keep the process alive and retrieve
every future's result so failures are observed.

Stop scheduled commit helpers and verify their last commit before job exit.

## Inference and Spaces quick reference

### Distinguish routing from deployment

`InferenceClient(..., provider="auto")` chooses an available provider for a
supported model and task under current routing rules. It does not create or
identify a dedicated Inference Endpoint.

Choose a named provider when processor, region, isolation, scaling, billing,
or optional chat features matter. To use a dedicated deployment, target its
endpoint URL explicitly.

Hub-routed inference can use a Hugging Face token with the required inference
permissions and billing association. A direct partner route uses that
provider's documented key. Never send a partner key to an arbitrary model
repository URL.

### Wait for endpoint state

Endpoint create and update operations are asynchronous. Poll the returned
remote state and handle terminal failure before directing traffic to it.

- `scale_to_zero` keeps configuration and allows a later request to cold-start.
- `pause` requires an explicit resume.
- Endpoint exposure is configured independently of source-repository privacy.

### Make Space state explicit

Ordinary Space filesystem data is ephemeral across restarts and rebuilds.
Store durable data on provisioned persistent storage at its documented mount,
or in an external service.

A sleeping Space wakes on access. A paused Space requires an explicit restart
or resume. Restarting does not promise preservation of ephemeral files.

README `suggested_hardware` and `suggested_storage` recommend settings; they do
not allocate resources. Space variables are readable to users with settings
access, while secrets become write-only in the settings interface after
creation. OAuth configuration for user login does not automatically authorize
the server process to read private repositories.

## Review checklist

- Confirm the installed Python requirement and API signatures.
- Remove legacy client classes, CLI names, arguments, and transfer switches.
- Configure HTTPX clients and catch transport versus Hub response errors.
- Use explicit token behavior and safe redirect handling.
- Guard sensitive writes with `parent_commit` where supported.
- Copy central-cache files before editing and clean cache layers deliberately.
- Preserve resumable-upload state and await process-local background work.
- Distinguish routed inference, partner credentials, and dedicated endpoints.
- Poll endpoint operations and provision Space persistence explicitly.
