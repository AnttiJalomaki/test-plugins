# Server Configuration and Operations

## Exact and literal environment values

`KC_` environment values undergo expression evaluation: `${...}` is resolved
and `$$` collapses to `$`. Use the equivalent `KCRAW_` name to preserve dollar
characters exactly. Defining both forms for the same key is a startup error.

```bash
export KCRAW_DB_PASSWORD='my$$pa${vault}word'
```

Environment-key normalization cannot represent every option name, especially
logging categories containing underscores. Pair an arbitrary `KC_` value
variable with a same-suffix `KCKEY_` variable that supplies the exact key.

```bash
export KC_MYKEY=debug
export KCKEY_MYKEY=log-level-package.class_name
```

## Optimized builds and providers

Every build option is persisted in plaintext, including options sourced from a
Java KeyStore. Never place secrets in build options. Under `start --optimized`,
a repeated build option is ignored when it matches the built value and rejected
when it differs. Run another build to change it.

```bash
bin/kc.sh build --db=postgres
bin/kc.sh start --optimized
```

Docker can change provider JAR modification times between build and runtime,
making startup report a changed provider. Set deterministic timestamps before
the optimized build.

```dockerfile
ADD --chown=keycloak:keycloak --chmod=644 some-jar.jar /opt/keycloak/providers/
RUN touch -m --date=@1743465600 /opt/keycloak/providers/*
RUN /opt/keycloak/bin/kc.sh build
```

## Request queues and bootstrap readiness

The HTTP request queue is unlimited by default. Set
`http-max-queued-requests` to cap waiting requests; excess work receives HTTP
503 immediately.

```bash
bin/kc.sh start --http-max-queued-requests=1000
```

With health endpoints enabled, the HTTP(S) and management endpoints open while
initialization continues. Startup and liveness can be UP while readiness is
DOWN. Route traffic using `/health/ready`. Alternatively, set
`--server-async-bootstrap=false` to keep endpoints closed until initialization
finishes.

## Datasources and health

Exclude individual optional additional datasources from health checks when
their failure must not make the entire deployment unhealthy. (26.7.0)

PostgreSQL transactions that touch only ephemeral session, login-failure, or
event tables use asynchronous commit; logout stays synchronous. Disable this
with `--spi-connections-jpa--quarkus--async-commit=false`. (26.7.0)

## Truststores and strict FIPS

Generated system truststores sourced from `conf/truststores` or
`--truststore-paths` use BCFKS in strict FIPS mode so BCFIPS can load them in
approved mode. Default and non-strict FIPS deployments continue to use PKCS12
when supported. (26.7.0)

## Multi-cluster v2

Enable preview multi-cluster v2 with `stateless`. This design removes the
external Infinispan cluster and fencing infrastructure. Nodes connect directly
using embedded caches, use the synchronously replicated database as the source
of truth, and distribute invalidations through a database-backed outbox.
(26.7.0)

## Operator installation

Install the Operator declaratively on vanilla Kubernetes using kustomize rather
than applying its component manifests separately. (26.7.0)

Preview cluster-wide mode lets one Operator reconcile `Keycloak` resources in
all namespaces. Choose OLM `AllNamespaces` install mode or, for non-OLM
installations, the `cluster-wide` kustomization overlay. (26.7.0)

## Graceful shutdown

The default shutdown timeout is ten seconds, and clustered nodes also wait for
cache rebalance. Roll changes one node at a time. Set `shutdown-timeout=1s`
only when the former one-second behavior is deliberate. (26.7.0)
