# Security and Support Policy

Use this reference to assess upgrade urgency, compatibility guarantees, provider
risk, controller privilege, deployment hardening, and release identity.

## Support window

ESO supports only its newest minor release. Publishing a new minor immediately
deprecates the prior minor. At the operations-and-security snapshot, 2.8 is the
supported line, guarantees Kubernetes 1.35, and reaches end of life when 2.9 is
published. Image rebuilds, Go dependency updates, and security and bug fixes are
applied to the supported line. Upgrade one minor at a time.

This short window means a pinned controller is not a long-term supported branch.
Plan CRD, controller, chart, and policy validation as a recurring minor-upgrade
operation.

## Deprecation guarantees are narrow

The protected compatibility surface consists of (operations-and-security):

- API object specs, status, and conditions;
- enums and constants;
- controller flags and environment variables;
- metrics;
- documented `ExternalSecret` update mechanics.

Helm charts, releases, images, signatures, OLM builds, source imports, and
unspecified behavior are outside that guarantee. Introducing a deprecation
requires a minor release during 0.x and a major release from 1.x onward. Only
removals from the protected surface inherit Kubernetes deprecation timelines.

ESO's component policy still classifies it as beta. Beta features are enabled by
default and considered safe to enable, but schemas or semantics may change
incompatibly with migration instructions; the policy does not recommend beta
software for production.

## Provider maturity is not feature parity

The operations-and-security support table classifies these providers as stable:

- AWS Secrets Manager and AWS Parameter Store;
- Akeyless;
- Azure Key Vault;
- CyberArk Secrets Manager;
- GCP Secret Manager;
- HashiCorp Vault;
- IBM Cloud Secrets Manager;
- Oracle Vault;
- Previder.

Kubernetes and SecretServer are beta. Every other provider in that table is
alpha. A maturity label does not promise the same operations: find, metadata
fetch, referent authentication, store validation, push, merge, and delete support
vary independently. Consult the provider references and its live CRD schema
before designing a workflow.

All providers have build tags, so unwanted providers can be excluded from a
custom binary (1.1.0).
Alibaba and Device42 were removed in 2.0.0 because they were unmaintained and
unsupported; no maturity label or compatibility promise keeps a removed provider
available.

## Pod defaults versus chart hardening

Default pod security contexts have used the restricted profile since 0.8.2
(operations-and-security):

- non-root UID `1000`;
- read-only root filesystem;
- privilege escalation disabled;
- all Linux capabilities dropped;
- `RuntimeDefault` seccomp.

Those pod settings do not make the chart a hardened deployment. NetworkPolicy
and metrics TLS/authentication default off. Blanket ServiceAccount token creation
and RBAC aggregation into view, edit, and admin roles default on. Render values,
inspect the resulting RBAC, and opt into the controls described in
`helm-and-operations.md`.

## Namespace and identity boundaries

Namespaced resources cannot cross-reference namespaced stores, Secrets, or other
namespaced referents. Use a namespace-scoped installation when cluster fan-out is
not required, and explicitly review every cluster-scoped controller and
`ClusterSecretStore` condition when it is required (operations-and-security).

Provider `serviceAccountRef` authentication needs TokenRequest authority. A
broad `serviceaccounts/token` grant lets the controller request tokens for every
ServiceAccount in scope. Disable the broad chart rule and grant creation only for
the named provider accounts. See `helm-and-operations.md` for the Role and
RoleBinding pattern.

## Generic targets expand privilege

`genericTargets.enabled` defaults to false because enabling it grants the
controller create, update, and delete access to ConfigMaps plus the configured
verbs for every additional type in `genericTargets.resources`
(operations-and-security). Treat every added API group and resource as a direct
authority expansion. Provide encryption, admission controls, and audit policy
appropriate to each target; a resource being template-compatible does not make
it secret-safe.

## Network and serving posture

Constrain egress to the Kubernetes API, DNS, and the selected provider endpoints;
prefer private endpoints. Constrain ingress to the documented controller,
webhook, and cert-controller ports. Policy controls should also deny unused
providers, restrict remote-key prefixes, and limit which namespaces can use
cluster stores (operations-and-security).

Metrics secure serving arrived in 0.20.0, authentication and authorization
through `FilterProvider` in 2.5.0, optional chart NetworkPolicy in 2.8.0, and
HTTP/2 can be disabled as of 0.20.0. These are separate controls: enable and
validate each one needed by the threat model.

## Availability is part of the security posture

Controller, webhook, and cert-controller replica, probe, PDB, and leader-election
defaults do not provide full high availability. Enable the controls deliberately
and assign distinct lease IDs to independent deployments in one namespace
(operations-and-security). Availability settings and their release-specific
behavior are detailed in `helm-and-operations.md`.

## Release artifact identity

ESO images publish keyless Cosign signatures, SLSA provenance attestations, and
SPDX JSON SBOM attestations (operations-and-security). Verify an immutable image
digest and validate both:

- certificate issuer: `https://token.actions.githubusercontent.com`;
- certificate subject: the External Secrets release workflow on
  `refs/heads/main`.

Signatures and provenance are themselves outside the deprecation guarantee.
Verification proves the checked artifact's release identity; it does not expand
the compatibility or support policy.
