# Client migration and packaging

## Runtime compatibility

The v1 client migration guidance in this reference is attributed to
`huggingface-hub-v1-client`.

Version 1.0 initially requires Python 3.9 or newer. Later 1.x releases may
raise the minimum, so derive compatibility from the installed release's
package metadata instead of treating Python 3.9 as a permanent 1.x floor.

## HTTP client migration

The client replaced both `requests` and `aiohttp` with HTTPX. Update code at
all of these boundaries:

| Old assumption | Current behavior |
| --- | --- |
| Transport exceptions come from `requests` or `aiohttp` | Transport failures are based on `httpx.HTTPError` |
| Hub response failures use a generic transport exception | Hub responses use the `HfHubHttpError` hierarchy |
| `configure_http_backend` installs a backend | Use `set_client_factory` or `set_async_client_factory` |
| Calls accept `proxies=` | Configure proxies on a global or custom HTTPX client |

Configure TLS, timeouts, transports, and mocks through the HTTPX client as
well. Audit exception handlers and test fixtures in addition to request calls;
otherwise an application can compile while no longer catching or simulating
the correct failure type.

## Removed abstractions

Three old client abstractions are gone:

- Replace `Repository` with `HfApi` or supported Git/Xet tooling for
  repository work.
- Replace `HfFolder` with `login`, `logout`, `auth_switch`, and `get_token`.
- Replace `InferenceApi` with `InferenceClient` or
  `AsyncInferenceClient`.

Moving from `Repository` to commit-oriented HTTP operations is not only a
rename. Re-evaluate conflict handling, atomicity boundaries, and any behavior
that depended on a local Git worktree.

## Command-line migration

The `huggingface-cli` executable was replaced by `hf`. Update scripts and
runbooks to the current command families, such as:

```bash
hf auth
hf download
hf upload
hf cache
```

Do not leave an old executable name hidden in CI images, shell wrappers, or
authentication instructions.

## Authentication argument migration

The `use_auth_token` alias was removed. Code and downstream wrappers must
expose and pass `token`:

```python
api.model_info("org/model", token=token)
```

When adapting a wrapper, change its public argument rather than translating an
indefinitely retained compatibility alias.

## Download argument migration

Downloads no longer accept:

- `resume_download`
- `force_filename`
- `local_dir_use_symlinks`

Supported cache behavior handles resumption where applicable. Use returned
paths, `force_download`, the central cache, and current `local_dir` behavior
instead of recreating the removed controls.

```python
snapshot_download("org/model")
```

Do not infer a forced filename by concatenating a directory with an expected
name; consume the returned path.

## Upload return values

Upload methods return commit-oriented information or URLs rather than the old
file-CDN abstraction. Follow the exact return type documented for the
installed v1 operation.

In particular, remove code that depends on:

- truthiness of the old result object;
- string concatenation with an assumed CDN path; or
- local Git-wrapper semantics that are not part of an HTTP commit result.

## Xet-first transfers

Xet is the supported large-file transfer path and integrates automatically
through `hf_xet`. The `hf_transfer` integration was removed, so
`HF_HUB_ENABLE_HF_TRANSFER` no longer enables it.

`HF_XET_HIGH_PERFORMANCE` is the relevant high-performance switch. Enable it
only when its documented resource tradeoff is appropriate for the host.

## Framework and dependency boundaries

Deprecated TensorFlow and Keras helpers were removed from the core client.
Framework integrations should own serialization and model-card callbacks,
then call Hub primitives for remote operations.

Optional and CLI dependency groups changed with v1. Packaging should select
the extra or direct dependency actually supported by the installed release;
do not assume a historical extra still installs the required framework or CLI
components.
