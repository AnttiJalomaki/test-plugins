# Gateway, Ingress, and Shim Controllers

Use this reference when cert-manager generates Certificates from Ingress,
Gateway API, or ListenerSet resources, or when configuring HTTP-01 through
those resources.

## Generated Certificate annotations

### Copy arbitrary selected annotations

Since `1.18`, the controller option `--extra-certificate-annotations` accepts
annotation keys to copy from an Ingress or Gateway to its generated
Certificate. Limit the allowlist to metadata that downstream policy or issuer
integrations actually require.

### Alternative names

In `1.21`, cert-shim controllers process `cert-manager.io/alt-names` and
`cert-manager.io/ip-sans` on ingress-like resources and place those identities
on generated Certificates.

### Timing changes

In `1.20`, changes to the Duration or `RenewBefore` annotation on an Ingress or
Gateway API resource immediately update the generated Certificate.

## Gateway TLS listener selection

### Passthrough listeners

Since `1.18`, TLS listeners with mode `Passthrough` are ignored rather than
being treated as certificate-issuing listeners.

### Exclude selected TLS listeners

In `1.21`, add `cert-manager.io/ignore-tls-listeners` to a Gateway to exclude
selected TLS listeners from certificate management. Gateway integration can
also be configured to consider listener protocols beyond its default set.
Keep the listener selection and protocol policy aligned so an intentionally
ignored listener is not reached through a broader protocol configuration.

## ListenerSet integration

### Enable certificate generation

In `1.20`, Gateway API integration can generate Certificates from annotated
ListenerSet resources. This is alpha and disabled by default; enable the
`ListenerSet` feature gate only after installing compatible Gateway API CRDs.

### Parent references

ACME Gateway configuration can leave Issuer or ClusterIssuer `parentRefs`
empty for inference. Certificate annotations can override the configured
references. Use 1.20.1 or later when combining issuer configuration and
annotation overrides because 1.20.0 can create duplicate `parentRef` entries.

### HTTP listener fallback

In `1.21`, a TLS-only ListenerSet can share its parent Gateway's HTTP listener
for HTTP-01. Add:

```yaml
metadata:
  annotations:
    acme.cert-manager.io/http01-parentreffallback: "true"
```

The generated solver HTTPRoute then refers to the parent Gateway.

## Gateway HTTP-01 for IP subjects

In `1.20`, the Gateway solver sets `HTTPRoute.spec.hostnames` when the challenge
DNS name is an IP address. This prevents generation of an invalid HTTPRoute for
an IP-address Certificate.

## Per-resource HTTP-01 ingress class

In `1.20`, an Ingress can override the configured solver's
`http01.ingress.ingressClassName` with:

```yaml
metadata:
  annotations:
    acme.cert-manager.io/http01-ingress-ingressclassname: nginx
```

This override is local to that Ingress. The solver itself must still obey the
rule that only one of `class`, `ingressClassName`, and `name` may be set.

## Controller configuration migration

In `1.21`, the flat controller fields `enableGatewayAPI` and
`enableGatewayAPIListenerSet` are deprecated. Migrate to the nested form; the
old fields continue to work during migration:

```yaml
gatewayAPI:
  enabled: true
  enableListenerSet: true
```
