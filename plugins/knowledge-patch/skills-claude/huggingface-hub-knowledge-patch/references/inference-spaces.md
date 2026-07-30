# Inference, endpoints, and Spaces

Use this reference when selecting inference routing, supplying credentials,
operating dedicated endpoints, configuring Space metadata, provisioning
persistence, or separating runtime secrets from OAuth.

## Routed providers are not dedicated deployments

`InferenceClient(..., provider="auto")` selects an available provider for a
supported model and task under current Hub routing rules.

```python
from huggingface_hub import InferenceClient

client = InferenceClient("org/model", provider="auto", token=token)
```

This routed surface is distinct from:

- serverless availability shown on a model page;
- a dedicated Inference Endpoint;
- a particular partner's direct API.

The common client surface does not guarantee identical processor type, region,
isolation, scaling, billing, or optional chat features across routes.

Select a named provider when those constraints matter. When the workload must
reach a dedicated deployment, explicitly target the deployed endpoint URL
rather than relying on `provider="auto"`.

## Credentials follow the route

Hub-routed inference can use a Hugging Face token carrying the required
inference permissions and billing association.

A direct partner-provider route uses that provider's key as documented for the
route. Keep these credential classes separate:

- do not assume a Hub token authenticates every direct provider route;
- do not send a partner credential to an arbitrary model repository URL;
- scope, store, rotate, and bill each credential according to the authority
  that issued it.

Resolve the destination before attaching credentials, especially when code
allows a model identifier, routed provider, or endpoint URL to be selected
dynamically.

## Dedicated endpoint operations are asynchronous

Creating or updating an Inference Endpoint returns before the remote operation
is necessarily ready for traffic.

A controller should:

1. Submit the create or update request.
2. Poll the returned remote state.
3. Continue through documented transitional states.
4. Detect and surface terminal failure.
5. Direct traffic only after a ready state is confirmed.

Do not infer readiness from a successful request submission.

## Scale-to-zero and pause differ

`scale_to_zero` retains endpoint configuration and permits a later request to
cold-start serving. Account for cold-start latency and client timeout behavior.

`pause` requires an explicit resume before serving. A normal inference request
should not be expected to resume it.

Endpoint exposure is configured independently of whether the source model
repository is private. Review endpoint authentication and network exposure
directly instead of deriving them from repository visibility.

## Space hardware and storage suggestions

README metadata fields `suggested_hardware` and `suggested_storage` recommend
choices to someone duplicating or configuring a Space. They do not allocate
hardware or persistent storage.

Actual resources must be selected through runtime settings. Code should not
infer that suggested storage exists or is mounted.

## Preload selected Hub files

`preload_from_hub` can stage narrowly selected Hub files during build or
startup. Pin revisions where reproducibility matters and keep the selection
narrow enough to control build size and time.

Preloading does not replace dependency declarations. Runtime libraries,
system packages, and other build requirements still belong in their documented
dependency mechanisms.

Custom HTTP headers for this facility are limited to the documented allowlist.
Do not assume arbitrary headers or credentials will be forwarded.

## Space filesystem persistence

The ordinary filesystem of a Space is ephemeral across restarts and rebuilds.
Durable state belongs:

- on separately provisioned persistent storage at its documented mount; or
- in an external durable service.

Do not store the sole copy of uploads, indexes, job checkpoints, or application
state on an ordinary runtime path.

Restarting a Space does not guarantee preservation of ephemeral files, even
when files appeared to survive a previous runtime transition.

## Sleeping and paused states

A sleeping Space wakes on access. Clients may need to tolerate wake-up delay.

A paused Space requires an explicit restart or resume; access alone does not
wake it. Operational checks should distinguish sleep, pause, build failure, and
application failure rather than treating all unavailable states alike.

## Variables and secrets

Space variables are visible to users with settings access. Use them for
non-sensitive configuration.

Secrets become write-only through the settings UI or API after creation. Both
variables and secrets are normally injected into the runtime as environment
variables, so application code must still prevent accidental logging,
diagnostic exposure, or propagation to child processes.

The write-only settings presentation does not make a secret invisible to the
running application that receives it.

## OAuth is a separate authority

README `hf_oauth` configuration can provision OAuth callback and client
settings plus requested scopes for user login.

That user-facing OAuth configuration does not automatically authorize the
Space's server process to access private repositories. Server-side repository
access needs its own deliberately provisioned credential and appropriate
permissions.

Keep these questions separate:

- Who can sign in to the Space?
- Which user scopes are requested?
- Which credential does the server process use?
- Which private repositories can that server credential access?

## Deployment checklist

- Decide whether the target is Hub routing, a named partner, or a dedicated
  endpoint URL.
- Attach only the credential type expected by that route.
- Poll endpoint create and update state through readiness or terminal failure.
- Treat scale-to-zero as cold-startable and pause as explicitly resumable.
- Configure endpoint exposure independently of repository visibility.
- Allocate actual Space hardware and persistent storage in runtime settings.
- Pin and narrow preloaded files; declare dependencies separately.
- Use only allowed custom headers.
- Put durable state on persistent storage or an external service.
- Distinguish sleeping and paused behavior.
- Keep public configuration, secrets, user OAuth, and server repository
  authority separate.
