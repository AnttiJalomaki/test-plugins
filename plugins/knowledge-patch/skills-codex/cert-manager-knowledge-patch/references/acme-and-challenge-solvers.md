# ACME and Challenge Solvers

Use this reference for ACME account and certificate behavior, HTTP-01 and
DNS-01 solver configuration, self-checks, retries, and challenge resources.

## Certificate profiles and renewal information

ACME issuers can select a certificate profile offered by the CA (since 1.18).
For example, Let's Encrypt offers `tlsserver` for ordinary server certificates
and `shortlived` for six-day certificates.

Experimental RFC 9773 ACME Renewal Information support is available in 1.21
behind `ACMEUseARI`. It queries the server's `renewalInfo` endpoint so the CA
can recommend a renewal window, including during mass revocations or CA key
rollovers.

Created Let's Encrypt account-key resources carry
`app.kubernetes.io/managed-by: cert-manager` from 1.18.

## HTTP-01 ingress behavior

Solver Ingresses use `PathType: Exact` from 1.18. ingress-nginx strict path
validation can reject this combination. From cert-manager 1.18.1, restore the
old path type with:

```yaml
config:
  featureGates:
    ACMEHTTP01IngressPathTypeExact: false
```

Other options are disabling ingress-nginx `strict-validate-path-type` or using
ingress-nginx 1.12.6+ or 1.13.2+.

A solver is rejected from 1.19 if more than one of `class`,
`ingressClassName`, and `name` is set. Select exactly one ingress mechanism.

An individual Ingress can override the solver's
`http01.ingress.ingressClassName` with the 1.20 annotation:

```yaml
metadata:
  annotations:
    acme.cert-manager.io/http01-ingress-ingressclassname: nginx
```

HTTP-01 challenges accept IPv6 addresses in the Host header from 1.18.5,
allowing IP-address certificates for IPv6 subjects.

## HTTP-01 solver Pods and resources

An Issuer or ClusterIssuer can set requests and limits for its own HTTP-01
solver Pods from 1.19. These values override the platform-wide
`--acme-http01-solver-resource-*` flags for that solver.

```yaml
spec:
  acme:
    solvers:
      - http01:
          ingress:
            podTemplate:
              spec:
                resources:
                  requests:
                    cpu: 20m
                    memory: 32Mi
                  limits:
                    cpu: 100m
                    memory: 64Mi
```

Runtime classes are configurable for HTTP-01 solver Pods in 1.21. For
chart-wide solver configuration:

```yaml
acmesolver:
  runtimeClassName: gvisor
```

The `--acme-http01-solver-extra-labels` controller flag added in 1.21 allows
Helm `global.commonLabels` to be propagated to dynamic HTTP-01 Pods, Services,
Ingresses, and Gateway API HTTPRoutes.

## Delayed validation and retries

HTTP-01 and DNS-01 solvers in 1.21 can use `waitInsteadOfSelfCheck` to skip the
cert-manager self-check, wait for a configured duration, and then request ACME
server validation. Treat this as an escape hatch for split-horizon DNS and NAT
hairpin environments.

Starting in 1.17.3, ACME authorization waits up to two minutes, reducing
premature `error waiting for authorization` failures.

In 1.21, TLS handshake timeouts, DNS failures, and context cancellation while
fetching an ACME nonce or waiting for authorization retry through workqueue
backoff instead of terminally failing the Challenge.

## DNS-01 solver controls

RFC2136 solver configuration accepts a `protocol` from 1.19, allowing an
explicit DNS update transport:

```yaml
spec:
  acme:
    solvers:
      - dns01:
          rfc2136:
            protocol: TCP
```

Azure DNS private zones, tenant selection, and provider-specific retry fixes
are described in the issuers and providers reference.

## Challenge status and permissions

The 1.19 metric `certmanager_certificate_challenge_status` exposes challenge
status for dashboards and alerts.

From 1.19.6, the aggregate `cert-manager-edit` ClusterRole no longer allows
creating Challenges or creating, patching, or updating Orders. Certificate-led
issuance is unaffected. Direct resource-management tools need explicit RBAC.

The `global.rbac.disableHTTPChallengesRole` Helm value existed only from 1.18.0
through 1.18.1 and was removed in 1.18.2 due to a bug. Do not depend on it.
