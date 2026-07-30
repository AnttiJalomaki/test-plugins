# Apollo Router Connectors, coprocessors, Rhai, and plugins

## Connector adoption and versioning

Apollo Connectors became generally available in Router `2.0.0`. Deployments using
the preview form should follow the GA schema upgrade path.

The default Connector specification changes by Router release. Router `2.14.0`
resolves an unversioned “latest/default” link to `connect/v0.3`; an explicit
`connect/v0.2` link remains on v0.2.

From `2.16.0`, linking to `https://specs.apollo.dev/connect/v0.4` is sufficient to
opt in. `connectors.preview_connect_v0_4` is a deprecated no-op.

`connectors.subgraphs` is deprecated in `2.15.0`; rename it to
`connectors.sources` before Router 3.

## Connector mapping language

### URI expressions (`2.2.0`)

Expressions can appear anywhere in or after a URI path, including query parameter
names. Expression results remain percent-encoded. Literal `[` and `]` are no
longer encoded unless invalid in a URI, and trailing slashes are preserved. Some
placements require Federation 2.11+.

```graphql
@connect(http: { GET: "/users?{$args.filterName}={$args.filterValue}" })
```

### Response content types (`2.3.0`)

Connector responses ending in `/json` or `+json` are parsed as JSON. `text/plain`
is decoded as a UTF-8 string available as `$`. Other content types become JSON
`null`; a missing header still assumes JSON. Deserialization failure produces
`CONNECTOR_DESERIALIZE` with `Response deserialization failed`.

Variables work inside nested input arguments from `2.3.0`.

### JSON parsing (`2.14.0`)

`->jsonParse` converts a JSON string into a structured value for immediate
selection. Non-string and invalid-JSON inputs fail; inferred shape is `unknown`.

```text
payload->jsonParse { users { name } }
```

### Connect v0.4 syntax (`2.15.0`)

Nested selections accept commas, object-property shorthand is valid, and
top-level object literals do not need `$()`. A primitive after `name:` is a literal
rather than a `$` lookup; qualify REST keys explicitly, especially names invalid
in GraphQL. The v0.2 and v0.3 parsers are unchanged.

```text
{ id, address { street, city }, next: $."@odata.nextLink" }
```

String methods include:

- `->split(separator[, limit])`; the separator may come from data, empty separators
  split into UTF-8 characters, and `limit` caps result count.
- `->trim`, `->trimStart`, and `->trimEnd`, which remove Unicode whitespace.

All reject non-string inputs.

An `@connect` may omit `http` and resolve a field, including a nested mutation, by
applying `selection` to arguments or enclosing-object data. A requestless mapping
cannot use response body data, `$status`, or `$response`; composition rejects such
transport-derived references.

### v0.4 validation and migration (`2.16.0`)

Composition recognizes fields beneath list-producing arrow methods such as
`->entries` and accepts nested scalar-list projections such as
`data->map(@->map(@->toString))`.

Self-referential Connector input types compose without an infinite schema walk;
shape inference stops the cycle at an unknown shape.

The separate `connect-migrate` tool evaluates selections under their linked
version and v0.4, classifying deterministic `$.` rewrites, unchanged expressions,
and cases needing judgment. It is not in the Router runtime; build it from
`apollo-federation` with the non-default `connect-migrate` Cargo feature.

## Connector traffic, TLS, and headers

### Traffic shaping (`2.1.0`)

Target all Connectors with `traffic_shaping.connector.all`, or a source with
`traffic_shaping.connector.sources` keyed by `subgraph_name.source_name`.
Connector traffic shaping does not support `deduplicate_query`.

```yaml
traffic_shaping:
  connector:
    all:
      timeout: 5s
    sources:
      connector-graph.random_person_api:
        global_rate_limit:
          capacity: 20
          interval: 1s
        timeout: 1s
```

### TLS (`2.1.0`)

Configure source-specific certificate authorities and mutual TLS under
`tls.connector.sources`:

```yaml
tls:
  connector:
    sources:
      connector-graph.random_person_api:
        certificate_authorities: ${file.ca.crt}
        client_authentication:
          certificate_chain: ${file.client.crt}
          key: ${file.client.key}
```

### Header propagation (`2.2.0`)

Use `headers.connector.all` or `headers.connector.sources`, keyed by
`<subgraph>.<source>`. Router YAML overrides headers defined through `@connect` or
`@source`.

```yaml
headers:
  connector:
    all:
      request:
        - propagate:
            named: x-client-header
    sources:
      connector-graph.random_person_api:
        request:
          - insert:
              name: x-inserted-header
              value: hello
```

## Coprocessors

### Validation and stage endpoints

Coprocessor GraphQL response validation defaults to enabled from `2.5.0` and
handles subscription termination responses. Disable at the coprocessor level only
when deliberately accepting nonconforming responses:

```yaml
coprocessor:
  response_validation: false
```

From `2.8.0`, router, supergraph, execution, and subgraph stages may each define a
`url` overriding the global URL.

### Connector stages and Unix sockets (`2.12.0`)

Colocated coprocessors may communicate over Unix sockets. They can also run at
`ConnectorRequest` and `ConnectorResponse`, receiving URI, headers, body, context,
and service identity as appropriate.

```yaml
coprocessor:
  url: http://localhost:3007
  connector:
    all:
      request:
        uri: true
        headers: true
        body: true
        context: all
        service_name: true
      response:
        headers: true
        body: true
        context: all
        service_name: true
```

Invalid non-UTF-8 header values now produce a warning naming the header while
valid headers continue through `externalize_header_map`; the entire conversion no
longer fails.

### Selective response bodies (`2.14.0`)

At supergraph, execution, and subgraph response stages, select `data`, `errors`,
and `extensions` independently. Boolean `body` remains accepted. A coprocessor may
modify only received fields; omitted fields retain original values.

```yaml
coprocessor:
  supergraph:
    response:
      body:
        data: false
        errors: true
        extensions: true
```

### Context and response conditions

Router `2.13.0` fixes the `context: true` merge regression that deleted returned
keys; the `context: deprecated` workaround is unnecessary.

In `2.16.0`, at parallel subgraph stages a response can delete only keys sent to
its stage, preventing deletion of keys concurrently added elsewhere.

Response-stage calls such as `on: response` can test request headers with
`exists: { request_header: x-name }` (`2.13.0`). The request-stage result is
retained for response-time evaluation.

## Rhai

Rhai can read and rewrite `request.uri.scheme` and
`request.subgraph.uri.scheme` (`2.1.0`):

```rhai
request.subgraph.uri.scheme = "https";
```

With `--hot-reload`, Rhai source edits trigger the same reload as configuration or
schema changes. Router `2.4.0` preserves multipart upload `Content-Type` through
Rhai processing, avoiding
`invalid multipart request: Content-Type is not multipart/form-data`.

`apollo.router.operations.rhai.duration` is a seconds histogram for every callback
(`2.14.0`); `rhai.stage` and `rhai.succeeded` identify location and outcome.

Rhai interns strings by default. Set `intern_strings: false` when high-concurrency
creation of new strings causes write-lock contention:

```yaml
rhai:
  scripts: ./rhai
  main: main.rhai
  intern_strings: false
```

## External services and transports

Subgraph endpoints can use Unix sockets from `2.13.0`. Put the request path in the
URL query, for example `unix:///tmp/some.sock?path=some_path`.

For subgraphs, Connectors, and coprocessors,
`experimental_http2: http2only` uses prior-knowledge h2c without TLS (`2.13.0`).
Plain `enable` without TLS remains HTTP/1.1 because no h2c upgrade is performed.

Trusted insecure graph-artifact registry hostnames can be allowlisted for HTTP
pulls (`2.13.0`), supporting private registries and pull-through caches.

Router releases may be downloaded through a proxy mirror when direct GitHub
access is unavailable (`2.1.0`).

## Correctness fixes and extension behavior

Router `2.1.2` preserves Federation `@context` and `@fromContext` when adding a
Connector. Router `2.4.0` fixes the 2.3 SigV4 regression that rejected some valid
configurations at startup.

When a Connector's `isSuccess` expression is false, configured
`errors.extensions` deep-merges with defaults (`2.16.0`). A custom nested `http`
object therefore retains default `http.status`.

Connector custom-instrument selectors are described in
`router-observability.md`; downstream response limits and connection metrics are
covered by the security and observability references.
