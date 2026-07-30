# Operations and observability

## OpenTelemetry changes

LiteLLM-specific error details moved under the `litellm.*` namespace, so update queries that use previous keys. Streaming spans add `gen_ai.response.time_to_first_chunk`; failed calls emit `gen_ai.client.operation.exception`; v2 error spans again expose `error.*` attributes.

## Database topology

`DATABASE_URL_READ_REPLICA` sends read-only Prisma operations to a reader while writes remain on `DATABASE_URL`. With `IAM_TOKEN_DB_AUTH=true`, tokens for both connections are refreshed.

`database_disable_prepared_statements` adds `pgbouncer=true`; `database_extra_connection_params` takes precedence. `supported_db_objects` restricts the stored object classes loaded, and `proxy_config_reload_interval_seconds` controls database-backed cross-pod convergence with a default of 30 seconds.

## Pool sizing and timeouts

`database_connection_pool_limit` applies per worker, so total capacity equals instances multiplied by workers multiplied by the configured limit. `database_connection_timeout` bounds a general connection call, `database_connect_timeout` bounds opening a connection, and `database_socket_timeout` bounds idle or silent sockets.

```yaml
general_settings:
  database_connection_pool_limit: 10
  database_connection_timeout: 60
  database_connect_timeout: 15
  database_socket_timeout: 300
```

## Drain and client disconnects

`enable_drain_endpoint` exposes `GET /health/drain` for pre-stop hooks and is disabled by default. Without `drain_endpoint_token`, it is unauthenticated; otherwise requests must carry the matching `X-Drain-Token`.

`cancel_on_disconnect: true` cancels non-streaming upstream work after a client disconnect and records status 499.

## Redis circuit breaker

The Redis circuit breaker is enabled by default. It opens after five consecutive failures and tries recovery after 60 seconds. Override with `REDIS_CIRCUIT_BREAKER_ENABLED`, `REDIS_CIRCUIT_BREAKER_FAILURE_THRESHOLD`, and `REDIS_CIRCUIT_BREAKER_RECOVERY_TIMEOUT`.

## Health and streaming stalls

`use_shared_health_check` stores deployment health in Redis for multi-instance deployments. `health_check_staleness_threshold` expires old results, while `health_check_ignore_transient_errors` excludes 408 and 429 probe failures from health-routing and cooldown decisions.

`ttft_timeout` detects a provider that never emits its first token and internally streams non-streaming calls. `stream_idle_timeout` detects token gaps. `LITELLM_MAX_STREAMING_DURATION_SECONDS` caps total stream lifetime; `LITELLM_STREAM_INACTIVITY_TIMEOUT_SECONDS` catches async streams that send keepalives without content.

## Runtime compatibility

Package metadata allows Python 3.14 with an upper bound of `<3.15`. Compatible `redisvl`, `pypdf`, `openapi-core`, and native-bridge dependencies are included for that runtime.

## Operational request bounds

`max_request_size_mb` rejects oversized requests. `max_response_size_mb` prevents oversized model responses from being sent. `pass_through_request_timeout` bounds custom and native-provider pass-through calls at 600 seconds by default; endpoint-specific timeouts override it.
