# Gateway, Ingress, and cert-shim

Use this reference for source annotations, listener selection, ListenerSets,
Gateway solver HTTPRoutes, and generated Certificate reconciliation.

## Listener behavior

Gateway TLS listeners in `Passthrough` mode are ignored from 1.18 rather than
being treated as certificate-issuing listeners.

The `cert-manager.io/ignore-tls-listeners` annotation added in 1.21 excludes
selected Gateway TLS listeners from management. Gateway integration can also be
configured to consider listener protocols outside its default set.

## ListenerSet integration

Certificate generation from annotated ListenerSet resources is alpha in 1.20,
disabled by default, and requires the `ListenerSet` feature gate.

For ACME Gateway configuration, issuer `parentRefs` may be left empty for
cert-manager to infer. Certificate annotations can override configured
references. Use 1.20.1 or later when combining issuer configuration with
annotation overrides because 1.20.0 can produce duplicate `parentRef` entries.

In 1.21, a TLS-only ListenerSet can direct a solver HTTPRoute to its parent
Gateway's HTTP listener:

```yaml
metadata:
  annotations:
    acme.cert-manager.io/http01-parentreffallback: "true"
```

## Gateway controller configuration

The flat controller fields `enableGatewayAPI` and
`enableGatewayAPIListenerSet` are deprecated in 1.21. They still work, but new
configuration should use the nested structure:

```yaml
gatewayAPI:
  enabled: true
  enableListenerSet: true
```

## Gateway HTTP-01 challenges

From 1.20, the Gateway solver sets `HTTPRoute.spec.hostnames` when the
challenge DNS name is an IP address. This avoids invalid HTTPRoutes for
IP-address certificates.

Common labels can reach dynamically generated Gateway HTTPRoutes through the
1.21 `--acme-http01-solver-extra-labels` controller flag.

## Ingress-specific solver selection

The 1.20 annotation below overrides the HTTP-01 solver's
`http01.ingress.ingressClassName` for one Ingress:

```yaml
metadata:
  annotations:
    acme.cert-manager.io/http01-ingress-ingressclassname: nginx
```

At the Issuer solver level, configure exactly one of `class`,
`ingressClassName`, and `name`; 1.19 rejects ambiguous multi-selection.

## Generated Certificate annotations

The controller option `--extra-certificate-annotations` accepts keys to copy
from Ingresses or Gateways into generated Certificates from 1.18.

Changing a Duration or `RenewBefore` annotation on an Ingress or Gateway
triggers immediate reconciliation of the generated Certificate from 1.20.

Cert-shim controllers process these annotations on ingress-like resources from
1.21:

```yaml
metadata:
  annotations:
    cert-manager.io/alt-names: www.example.com,api.example.com
    cert-manager.io/ip-sans: 192.0.2.10
```

## Deletion behavior

While a generated or directly authored Certificate is being deleted, the
Certificate controller does not create new CertificateRequest or Secret child
objects (since 1.17). Account for that behavior when debugging finalizers or
watching source-resource reconciliation.
