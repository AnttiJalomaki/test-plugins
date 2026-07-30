# Repository operations and authentication

## Guard writes with the expected branch head

Repository mutations that accept `parent_commit` can use it as an optimistic
concurrency guard. Pass the commit that was the known branch head when the
operation was prepared:

1. Read or retain the current head.
2. Construct the mutation against that state.
3. Submit the known head as `parent_commit`.
4. If the branch moved, handle the failure and re-evaluate the change.

Without this guard, a mutation can be applied on a base the caller did not
inspect. Use it whenever overwrites or lost updates would be unsafe.

## Protect bearer tokens across redirects

Repository `resolve` URLs can redirect to content-addressed storage. A raw
HTTP implementation must follow the redirect while ensuring that the bearer
token is not forwarded to an unrelated origin.

`hf_hub_download` and the supported client flow handle this safely. If a raw
client is required, treat cross-origin redirect authentication as an explicit
security boundary rather than enabling unrestricted credential forwarding.

## Token resolution

`HF_TOKEN` has precedence over the credential stored on disk.

For APIs with a `token=` parameter:

| Value | Meaning |
| --- | --- |
| credential string | Use that exact credential |
| `True` | Require the locally resolved token |
| `False` | Suppress authentication |

Set `HF_HUB_DISABLE_IMPLICIT_TOKEN=1` when an available token must not be
attached to otherwise-anonymous reads.

```python
api.model_info("open/model", token=False)
api.model_info("org/private-model", token=True)
```

Use explicit `False` for intentionally public access and explicit `True` when
local authentication is required. This avoids making authorization behavior
depend silently on whether a developer or runner happens to be logged in.

## Token file placement

`HF_TOKEN_PATH` overrides the location of the stored-token file beneath
`HF_HOME`. Cache variables are independent:

- `HF_HUB_CACHE` moves Hub cache content, not authentication state.
- `HF_XET_CACHE` moves Xet cache content, not authentication state.

When relocating an installation, configure the token path deliberately rather
than assuming cache relocation also moves credentials.

## Logout and remote revocation

`logout()` and `hf auth logout` remove saved local credentials. They do not
revoke the token on the Hub.

For a compromised or retired token, perform both actions:

1. remove the saved local credential; and
2. revoke the remote token in Hub settings.

Logout is also not data erasure. Private files already downloaded into local
cache or working directories remain on disk and require a separate cleanup
decision.
