# Observability and security

## Structured task logging

`LoggingMixin.log`, including hook and operator loggers, is a structlog logger
as of 3.1.0. Standard-library logging calls remain valid. Structlog calls can
attach searchable fields:

```python
self.log.info("Registering adapter", name=item.name)
```

## API server JSON logs

Set `[logging] json_logs` in 3.2.0 to emit API-server access logs, warnings,
exceptions, and other server output as newline-delimited JSON:

```ini
[logging]
json_logs = True
```

`airflow celery worker` does not yet support this JSON mode.

Other 3.2.0 controls include `log_timestamp_format` for component timestamps
and `uvicorn_logging_level` for API access-log verbosity. The Execution API
propagates correlation IDs across components, and the
`executor.running_dags` gauge counts running Dags.

## Remote log handler discovery

In 3.3.0, remote logging resolves lazily in this order:

1. A custom `[logging] logging_config_class`.
2. A provider `RemoteLogIO` selected by the `remote_base_log_folder` URI
   scheme.
3. The transitional `airflow_local_settings.py` fallback.

A provider handler needs a no-argument `from_config()` method.
`airflow.logging_config.load_logging_config()` is deprecated. Callback
subprocesses can upload remote logs, so include them when validating provider
handlers and credentials.

## OpenTelemetry metrics and tracing

OpenTelemetry timer metrics are Histograms rather than Gauges in 3.3.0.
Dashboards and alerts must expect the histogram contract.

`dag_processing.last_run.seconds_ago` now includes `file_path`, `bundle_name`,
and `file_name` tags. The legacy filename-suffixed metric stays enabled unless
`[metrics] legacy_names_on` is disabled.

Head sampling is supported. Custom samplers receive `dag_id`, `run_id`, and
`run_type` attributes, which can drive workflow-aware sampling decisions.

## API transport and connection isolation

API clients and servers support mutual TLS and private certificate authorities
as of 3.3.0. Configure credentialed CORS explicitly: wildcard origins are
rejected when credentials would make them unsafe.

Connection tests may run asynchronously on workers. This keeps Connection
access away from the API-server process and should be reflected in worker
network policy, logs, and troubleshooting.

## Operational validation checklist

- Parse newline-delimited logs and NDJSON API streams as separate JSON records.
- Preserve correlation IDs across Execution API clients and remote runtimes.
- Check remote-log upload from normal tasks, callbacks, and failure paths.
- Update metric types and tag dimensions before enabling new collectors.
- Verify that legacy metric names are intentionally enabled or disabled.
- Test mTLS trust chains and CORS behavior with the actual clients.
- Ensure asynchronous connection-test workers can reach only required targets.
