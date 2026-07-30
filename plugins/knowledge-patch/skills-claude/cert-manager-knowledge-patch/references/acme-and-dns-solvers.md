# ACME and DNS Solvers

Use this reference for ACME account behavior, HTTP-01 and DNS-01 solver
configuration, validation timing, workload resources, and retry behavior.

## ACME account and certificate behavior

### Certificate profiles

Since `1.18`, ACME issuance can select a certificate profile offered by the
CA. For example, Let's Encrypt exposes `tlsserver` for ordinary server
certificates and `shortlived` for six-day certificates. Confirm that the
configured ACME server advertises the chosen profile.

### Renewal Information

`1.21` includes experimental RFC 9773 support behind the `ACMEUseARI` feature
gate. When enabled, cert-manager queries the ACME server's `renewalInfo`
endpoint and uses CA-recommended renewal windows, including windows associated
with mass revocations or CA key rollovers.

### Managed account-key label

Let's Encrypt account-key resources created since `1.18` carry
`app.kubernetes.io/managed-by: cert-manager`. Use the label for ownership
queries and policy without assuming it appears on older, pre-existing Secrets.

## HTTP-01 solver Ingresses

### Exact path behavior

Since `1.18`, generated challenge Ingresses use `PathType: Exact`. This can
conflict with ingress-nginx strict path validation. Starting in 1.18.1, restore
`ImplementationSpecific` with:

```yaml
config:
  featureGates:
    ACMEHTTP01IngressPathTypeExact: false
```

Other options are disabling `strict-validate-path-type` in ingress-nginx or
using ingress-nginx 1.12.6+ or 1.13.2+.

### Select one Ingress mechanism

In `1.19`, an HTTP-01 solver is rejected if it specifies more than one of
`class`, `ingressClassName`, and `name`. Configure exactly one selection
mechanism for a solver.

### Per-Ingress class override

In `1.20`, an individual Ingress can override the solver's
`http01.ingress.ingressClassName`:

```yaml
metadata:
  annotations:
    acme.cert-manager.io/http01-ingress-ingressclassname: nginx
```

### IPv6 subjects

Starting in 1.18.5, HTTP-01 handles IPv6 addresses in the Host header, enabling
issuance for IPv6 IP-address subjects.

## Solver workloads

### Per-Issuer resources

In `1.19`, an Issuer or ClusterIssuer can override platform-wide
`--acme-http01-solver-resource-*` flags with Pod requests and limits for a
specific solver:

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

### Runtime class and common labels

In `1.21`, `acmesolver.runtimeClassName` configures the runtime class for
HTTP-01 solver Pods:

```yaml
acmesolver:
  runtimeClassName: gvisor
```

The `--acme-http01-solver-extra-labels` controller flag allows Helm
`global.commonLabels` to propagate to dynamically created solver Pods,
Services, Ingresses, and Gateway API HTTPRoutes.

## Validation and retries

### Authorization timeout

Starting in 1.17.3, ACME challenge authorization waits for up to two minutes,
reducing premature `error waiting for authorization` failures.

### Delayed validation without self-check

In `1.21`, HTTP-01 and DNS-01 solvers can set `waitInsteadOfSelfCheck` to skip
cert-manager's self-check, wait for a configured duration, and then request
validation from the ACME server. Use this only for environments such as
split-horizon DNS or NAT hairpin deployments where a valid challenge cannot
pass the local self-check.

### Transient failures use workqueue backoff

In `1.21`, TLS handshake timeouts, DNS failures, and context cancellation while
fetching an ACME nonce or waiting for authorization retry through workqueue
backoff instead of terminally failing the Challenge.

## DNS-01 providers

### Cloudflare

A Cloudflare API change broke DNS-01 issuance in the initial 1.17 release. Use
1.17.1 or later.

### RFC2136 transport

Since `1.19`, RFC2136 solver configuration accepts `protocol` so the DNS update
transport can be selected explicitly:

```yaml
spec:
  acme:
    solvers:
      - dns01:
          rfc2136:
            protocol: TCP
```

### Azure private zones

Since `1.20`, Azure DNS-01 supports private zones through `zoneType`:

```yaml
spec:
  acme:
    solvers:
      - dns01:
          azureDNS:
            zoneType: AzurePrivateZone
```

### CloudDNS cleanup

In `1.20`, CloudDNS challenge cleanup removes ACME TXT records even when the
DNS name has a large resource-record set.

### DigitalOcean retries and events

In `1.20`, DigitalOcean DNS-01 retries are regulated, and the complete DNS-01
error is attached to the Challenge as an event for troubleshooting.

### Credential readiness

In `1.21`, DNS issuer Secrets are validated before the Issuer is marked ready.
Secret mistakes are surfaced at readiness time rather than being silently
accepted until issuance.
