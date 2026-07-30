# Migration, Security, and Deployment

## Major-upgrade checklist

### Flags promoted to normal behavior

For the 3.0 migration (`3.0-migration`), remove
`promql-at-modifier`, `promql-negative-offset`,
`new-service-discovery-manager`, `expand-external-labels`, and
`no-default-scrape-port` from `--enable-feature`. External labels now expand
`$var` and `${var}`; undefined variables become empty and `$$` is a literal
dollar. Scrape target labels no longer gain ports derived from the scheme.

Use `--agent` in place of the `agent` feature flag and
`--web.enable-remote-write-receiver` in place of `remote-write-receiver`.
Automatic `GOMEMLIMIT` and `GOMAXPROCS` sizing is enabled; disable it only with
`--no-auto-gomemlimit` or `--no-auto-gomaxprocs`.

### Removed commands, flags, and bundled files

The `storage.tsdb.allow-overlapping-blocks`, `alertmanager.timeout`, and
`storage.tsdb.retention` startup flags are rejected (since 3.0.0). Example
JavaScript and templates for the console feature are no longer bundled, so
console deployments must mount or package their own files.

Alertmanager's v1 API configuration is unsupported. Run Alertmanager 0.16.0 or
later and replace explicit `api_version: v1` with `api_version: v2`
(`3.0-migration`).

### Persistent-data downgrade floor

The TSDB format prepared in 2.55 makes 2.55 the lowest release that can open a
data directory used by 3.x. Upgrade through 2.55 before the major jump. A
downgrade below 2.55 requires abandoning that persistent data
(`3.0-migration`). Experimental XOR2 and histogram start-timestamp block formats
create stricter downgrade boundaries; see the storage reference.

### Log pipeline compatibility

Prometheus uses `log/slog` output rather than the earlier `go-kit/log` shape
(`3.0-migration`). Update parsers that require `ts`, `caller`, or lowercase
levels so they accept `time`, `source`, and values such as `level=INFO`.

## Security requirements

### Required patch levels

Deploy at least 3.11.3 on the 3.11 line (`3.11.0`). It fixes:

- AzureAD remote-write `client_secret` disclosure through `/-/config`
  (CVE-2026-42151).
- Failure to enforce the declared-length limit for Snappy remote-read requests
  (CVE-2026-42154).
- Stored XSS through metric or label values in current and old UIs
  (CVE-2026-40179 and GHSA-fw8g-cg8f-9j28).

STACKIT service-discovery secrets are no longer exposed in plaintext through
`/-/config` in fixed 3.12 releases; STACKIT users should upgrade (`3.12.0`).

Prometheus 3.13.0 updates `sanitize-html` for CVE-2026-44990, so UI-exposing
deployments should use that release or later (`3.13.0`).

### Redirect credential stripping

HTTP clients strip authorization headers, basic and bearer credentials, OAuth2
credentials, and configured headers when a redirect changes host (since
3.13.0). This applies to scraping, remote read/write, alerting, and service
discovery, closing CVE-2025-4673 and CVE-2023-45289. Do not design a cross-host
redirect flow that depends on forwarding secrets.

### Authentication additions

OAuth2 HTTP clients can use the RFC 7523 section 3.1 JWT bearer grant (since
3.8.0). SigV4 accepts `use_fips_sts_endpoint` for FIPS-compliant AWS STS
endpoints (since 3.8.0) and an AWS `external_id` (since 3.11.0). Details specific
to Azure remote write are in the remote-storage reference.

## Containers and release artifacts

### Writable data directory

The container image's `/prometheus` directory is writable (since 3.3.0).

### Busybox and distroless variants

From 3.10.0, `-busybox` and `-distroless` image variants are published; the
unsuffixed image is still busybox. Distroless runs as UID/GID 65532 and has no
`VOLUME` declaration. Fix named-volume or bind-mount ownership before switching:

```text
docker run --rm -v prometheus-data:/prometheus alpine chown -R 65532:65532 /prometheus
docker run -v prometheus-data:/prometheus prom/prometheus:latest-distroless
```

Images are also published through GitHub Container Registry (since 3.13.0).
Third-party npm licenses are served by the binary at
`/assets/third-party-licenses.txt`; tarballs and images no longer contain
`npm_licenses.tar.bz2` (since 3.13.0).

### Platform target

The `aix/ppc64` compilation target is supported (since 3.12.0).

## Runtime and shutdown compatibility

Prometheus 3.9.1 fixes an Agent-mode startup crash present in 3.9.0 and restores
scrape relabel `keep` and `drop`; use 3.9.1 when either path matters (`3.9.0`).

The `/-/ready` endpoint again returns `X-Prometheus-Stopping` while in the
`NotReady` shutdown state (since 3.10.0). Shutdown-aware health checks may use
that header.

Concurrent Agent appends for one label set no longer create duplicate in-memory
series or duplicate WAL records (since 3.12.0).

## Promtool HTTP configuration paths

Relative paths inside the file supplied through `--http.config.file` resolve
from that file's own directory (since 3.13.0), not from an extra parent
directory. Adjust configurations that depended on the old traversal.
