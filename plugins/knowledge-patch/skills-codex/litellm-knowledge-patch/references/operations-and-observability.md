# Operations and observability

## OpenTelemetry attributes

LiteLLM-specific error details use the `litellm.*` namespace, so update
queries that use older keys. Streaming spans expose
`gen_ai.response.time_to_first_chunk`; failed calls emit
`gen_ai.client.operation.exception`; v2 error spans expose `error.*`
attributes again (since 1.93.0).

## Redis roles

Configure coordination Redis independently of the response cache. The usage
cache can be built from `REDIS_*` environment variables. The request
allowlist under `general_settings` is applied to LiteLLM globals.

The Redis circuit breaker is enabled by default. It opens after five
consecutive failures and attempts recovery after 60 seconds. Override these
values with `REDIS_CIRCUIT_BREAKER_ENABLED`,
`REDIS_CIRCUIT_BREAKER_FAILURE_THRESHOLD`, and
`REDIS_CIRCUIT_BREAKER_RECOVERY_TIMEOUT`.

## Graceful drain

`enable_drain_endpoint` exposes `GET /health/drain` for pre-stop hooks and is
off by default. Without `drain_endpoint_token`, the route is unauthenticated.
When a token is configured, callers must send the same value in
`X-Drain-Token`.

## Database pools and timeouts

`database_connection_pool_limit` applies per worker. Calculate total
connection capacity as instances multiplied by workers multiplied by the
configured limit.

The general connection-call timeout, connection-open timeout, and
idle/silent-socket timeout are independent:

```yaml
general_settings:
  database_connection_pool_limit: 10
  database_connection_timeout: 60
  database_connect_timeout: 15
  database_socket_timeout: 300
```

## Config discovery

Set `CONFIG_FILE_PATH` to start `litellm` from a mounted config without
passing `--config`.

Alternatively, set `LITELLM_CONFIG_BUCKET_NAME` and
`LITELLM_CONFIG_BUCKET_OBJECT_KEY` to load config from S3. Add
`LITELLM_CONFIG_BUCKET_TYPE=gcs` to use GCS.

## Python runtime

Package metadata supports Python 3.14 with an upper bound of `<3.15` (since
1.93.0). The distribution includes compatible `redisvl`, `pypdf`,
`openapi-core`, and native-bridge dependencies for that runtime.
