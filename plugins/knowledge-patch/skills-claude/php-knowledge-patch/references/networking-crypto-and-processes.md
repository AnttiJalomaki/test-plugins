# Networking, Crypto, and Processes

Use this reference for cURL, OpenSSL, LDAP, PCNTL, sockets, filtering, mail,
and extension validation that affects network or process boundaries.

## cURL capabilities, callbacks, and redirects

### cURL feature discovery (8.4.0)

`curl_version()` includes a `feature_list` associative array for every known
cURL feature, with booleans indicating runtime support. Prefer it to manually
decoding only the legacy feature bitmask.

### cURL pre-request hook (8.4.0)

`CURLOPT_PREREQFUNCTION` installs a callable that runs after connection
establishment but before sending the request. It must return
`CURL_PREREQFUNC_OK` to proceed or `CURL_PREREQFUNC_ABORT` to cancel.

### cURL debug callback (8.4.0)

`CURLOPT_DEBUGFUNCTION` receives the `CurlHandle`, a `CURLINFO_*` message type,
and the debug message string throughout a request. It cannot be combined with
`CURLINFO_HEADER_OUT`; both use the same libcurl facility.

### Persistent cURL shares and large input sizes (8.5.0)

cURL share handles may persist across PHP requests for safe connection reuse.
Use `CURLOPT_INFILESIZE_LARGE` instead of `CURLOPT_INFILESIZE` when the latter
is limited to a signed 32-bit size even on a 64-bit system.

### cURL redirect modes (8.5.0)

`CURLOPT_FOLLOWLOCATION` accepts `CURLFOLLOW_OBEYCODE` for stricter
redirect-code handling, `CURLFOLLOW_FIRSTONLY` to stop after the first
redirect, and `CURLFOLLOW_ALL` as the equivalent of `true`.

```php
curl_setopt($handle, CURLOPT_FOLLOWLOCATION, CURLFOLLOW_FIRSTONLY);
```

## Cryptography and hashing

### Modern OpenSSL keys (8.4.0)

OpenSSL supports x25519, ed25519, x448, and ed448 keys for key creation and
details, signing, and verification.

### OpenSSL-backed Argon2 hashing (8.4.0)

`PASSWORD_ARGON2` hashing is available when PHP uses OpenSSL 3.2 or later in
an NTS build.

### OpenSSL key derivation length (8.5-migration)

The `key_length` argument of `openssl_pkey_derive()` is deprecated because it
is either ignored or truncates the derived key, which can be unsafe.

## Signals, sockets, LDAP, and validation

### PCNTL validation and failures (8.4-migration)

Signal-mask and signal-wait APIs reject empty or non-integer signal lists,
invalid signal numbers or mask modes, and invalid timed-wait durations with
`TypeError` or `ValueError`. Runtime failures consistently return `false`,
never `-1`.

### Stricter extension input validation (8.5-migration)

`bzcompress()` validates block size 1–9 and work factor 0–250. Intl timezone
and locale operations, LDAP options, `pcntl_exec()` arguments and environment,
and POSIX limits reject documented invalid states with exceptions. SNMP
validates hosts, ports, timeouts, and retry values. Sockets validate ports,
hints, and multicast contexts. Tidy validates configuration key types and
rejects invalid or read-only settings.

## Filtering and mail

### Exception-based filtering (8.5.0)

`FILTER_THROW_ON_FAILURE` makes filter validation failures throw instead of
returning a failure value. It cannot be combined with
`FILTER_NULL_ON_FAILURE`; that combination throws `ValueError`.

```php
$id = filter_var($input, FILTER_VALIDATE_INT, FILTER_THROW_ON_FAILURE);
```

### Observable sendmail failures (8.5.0)

With the sendmail transport, `mail()` reports the actual sendmail error, warns,
and returns `false` when sending fails or the process terminates unexpectedly.

