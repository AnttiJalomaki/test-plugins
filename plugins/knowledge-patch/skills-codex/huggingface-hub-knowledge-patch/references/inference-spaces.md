# Inference routing, endpoints, and Spaces

## Routed providers and dedicated deployments

`InferenceClient(..., provider="auto")` selects an available provider for a
supported model and task according to current Hub routing rules:

```python
from huggingface_hub import InferenceClient

client = InferenceClient("org/model", provider="auto", token=token)
```

This routed, serverless availability is separate from a dedicated Inference
Endpoint. The shared client surface does not guarantee that routes use the
same:

- processor;
- region;
- isolation;
- scaling behavior;
- billing behavior; or
- optional chat features.

Select a named provider when route constraints matter. Explicitly target the
deployed endpoint URL when the application intends to use a dedicated
deployment.

## Route-specific credentials

Hub-routed inference can use a Hugging Face token with the required inference
permissions and billing association. A direct partner-provider route uses
that provider's key according to the route's documentation.

Credentials are not interchangeable merely because the client surface is
similar. Never send a partner credential to an arbitrary model repository
URL.

## Inference Endpoint lifecycle

Creating or updating an Inference Endpoint is asynchronous. The returned
object is not proof that the endpoint is ready for traffic.

After a create or update:

1. poll the remote state;
2. wait for the serving-ready state;
3. handle a terminal failure explicitly; and
4. only then direct traffic to the endpoint.

`scale_to_zero` retains configuration and allows a later request to trigger a
cold start. `pause` does not wake on request and requires an explicit resume.

Endpoint exposure is an independent setting. A private source model repository
does not by itself define whether the endpoint is publicly or privately
exposed.

## Space hardware and storage suggestions

The README fields `suggested_hardware` and `suggested_storage` recommend
choices to people duplicating or configuring a Space. They do not allocate
the resources.

Configure actual hardware and persistent storage through runtime settings.
Do not use the suggestion fields as evidence that a running Space has the
recommended resources.

## Preloading and custom headers

`preload_from_hub` can stage narrowly selected, revision-pinned Hub files
during build or startup. Pin the intended revision and select only the files
that should be staged.

Preloading is not dependency management; declare runtime and build
dependencies separately. Custom HTTP headers are limited to the documented
allowlist, so an arbitrary header cannot be assumed to reach the application.

## Persistent and ephemeral state

The ordinary Space filesystem is ephemeral across restarts and rebuilds.
Place durable state either:

- on separately provisioned persistent storage at its documented mount; or
- in an external service.

A sleeping Space wakes on access. A paused Space requires an explicit restart
or resume. In either case, restarting does not guarantee that files on the
ephemeral filesystem survive.

## Variables, secrets, and OAuth

Space variables can be viewed by users with settings access. Space secrets
become write-only through the settings UI or API after they are created. Both
are normally delivered to the runtime as environment variables.

Use secrets for values that should not remain readable through settings, but
still treat runtime environment access as sensitive.

README `hf_oauth` configuration can provision callback and client settings
and requested scopes for user login. That user-facing OAuth setup is a
separate authority from the Space server process: it does not automatically
authorize server-side access to private repositories.
