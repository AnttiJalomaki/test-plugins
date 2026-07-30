# Client migration and compatibility

Use this reference when upgrading Python code, HTTP customization, CLI
automation, download or upload calls, large-file transfer configuration, or
framework integrations. The client-specific guidance corresponds to the
included `huggingface-hub-v1-client` compatibility batch.

## Python support is release-specific

Version 1.0 originally requires Python 3.9 or newer. A later 1.x release may
raise that floor, so do not encode Python 3.9 as a guarantee for the entire
major series. Inspect the installed release's package metadata and propagate
that constraint to downstream projects and environments.

## HTTPX transport migration

The client replaced both `requests` and `aiohttp` with HTTPX. Update exception
handling and customization together:

- HTTP transport failures derive from `httpx.HTTPError`.
- Hub response failures use the `HfHubHttpError` hierarchy.
- `configure_http_backend` gives way to `set_client_factory` and
  `set_async_client_factory`.
- Proxies, TLS, timeouts, transports, and mocks belong on a global or custom
  HTTPX client.
- Per-call `proxies=` is no longer the customization path.

Code that formerly caught `requests` or `aiohttp` exceptions will not
necessarily catch HTTPX transport failures. Keep Hub error handling separate
when response metadata or Hub-specific diagnostics are needed.

When installing a factory, return a correctly configured synchronous or
asynchronous HTTPX client for that factory's contract. Centralized client
configuration also means a test mock or transport policy can affect every call
using that factory.

## Removed client abstractions

`Repository`, `HfFolder`, and `InferenceApi` were removed.

- Use `HfApi` or supported Git/Xet tooling for repository work.
- Use `login`, `logout`, `auth_switch`, and `get_token` for authentication.
- Use `InferenceClient` or `AsyncInferenceClient` for inference.

Moving from `Repository` to commit-oriented HTTP operations changes semantics,
not only names. Revisit:

- how conflicts are detected;
- whether a multi-file workflow is atomic;
- which commit or branch head an operation targets;
- whether code expects a local Git worktree;
- how local uncommitted state used to participate in the operation.

## CLI replacement

The executable is `hf`, replacing `huggingface-cli`. Current automation should
use command groups such as:

```bash
hf auth
hf download
hf upload
hf cache
```

Update scripts, subprocess calls, documentation, images, and entry-point
checks. Do not assume that swapping the executable name is enough if an old
subcommand or option also changed.

## Authentication argument migration

The `use_auth_token` alias was removed. Callers and downstream wrappers must
accept and pass `token`:

```python
api.model_info("org/model", token=token)
```

If a wrapper exposes its own compatibility surface, normalize there and call
the Hub API with `token=` only. The meanings of string, `True`, and `False`
token values are detailed in [repositories-auth.md](repositories-auth.md).

## Download argument migration

Downloads no longer accept:

- `resume_download`;
- `force_filename`;
- `local_dir_use_symlinks`.

Supported cache behavior performs resumption where applicable, without the old
toggle. Use the returned path rather than deriving an old forced filename.
Choose `force_download` when a fresh download is truly required.

Select storage behavior through the central cache or current `local_dir`
semantics. Do not recreate removed symlink controls in a wrapper without first
understanding the current cache and materialization behavior.

```python
from huggingface_hub import snapshot_download

path = snapshot_download("org/model")
```

## Upload return values and repository semantics

Upload methods now return commit-oriented information or URLs rather than the
old file-CDN abstraction. Check the exact return annotation and runtime object
for the installed release.

Migration must remove assumptions based on:

- truthiness of the former result;
- concatenating a result as though it were a CDN URL string;
- treating an upload as a local Git-wrapper operation;
- inferring atomicity or conflict behavior from the old abstraction.

Use explicit fields or the documented URL/commit value that the current method
returns.

## Xet-first large-file transfers

Xet is the supported large-file transfer path and integrates automatically
through `hf_xet`.

The `hf_transfer` integration was removed.
`HF_HUB_ENABLE_HF_TRANSFER` no longer activates it. Remove that dependency and
environment switch from deployment configuration.

`HF_XET_HIGH_PERFORMANCE` is the relevant opt-in when maximum Xet throughput is
desired, but only enable it when the documented CPU, network, disk, and other
resource tradeoffs fit the environment. It is not a compatibility alias for
the removed transfer switch.

## Framework and packaging integrations

Deprecated TensorFlow and Keras utilities were removed from the core client.
A framework integration should own:

- serialization and deserialization;
- framework-specific save and load behavior;
- model-card callbacks and lifecycle hooks;
- translation between framework objects and Hub file or commit primitives.

Call supported Hub primitives from that integration rather than expecting the
core client to retain removed framework helpers.

Optional and CLI dependency groups changed. Packaging must select the actual
supported extra for the installed release or declare the needed dependency
directly. Do not retain an extra name solely because it existed on an older
release.

## Migration audit

Search code and configuration for:

```text
requests
aiohttp
configure_http_backend
proxies=
Repository
HfFolder
InferenceApi
huggingface-cli
use_auth_token
resume_download
force_filename
local_dir_use_symlinks
hf_transfer
HF_HUB_ENABLE_HF_TRANSFER
```

For every match, decide whether it is executable integration code, a pinned
legacy environment, or stale documentation. Verify replacements against the
installed release rather than applying textual substitutions blindly.
