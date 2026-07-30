# Networking, Crypto, Patterns, and I/O

Use this reference for cURL requests, cryptographic capabilities, regular
expressions, stream behavior, shell history, mail delivery, and runtime
capability checks. The source items are attributed to `8.4-migration`, `8.4.0`,
`8.5-migration`, and `8.5.0`.

## cURL validation and discovery

`curl_multi_select()` throws `ValueError` for an invalid timeout range in
`8.4-migration`. Validate the timeout rather than expecting permissive
coercion or warning-only behavior.

Since `8.4.0`, `curl_version()` includes `feature_list`, an associative array
of every known cURL feature and a boolean indicating support. Prefer named
runtime capability checks over decoding only the legacy bitmask.

```php
$features = curl_version()['feature_list'];
```

`CURLOPT_BINARYTRANSFER` is deprecated in `8.4-migration`.

## cURL request callbacks

### Pre-request callback

Since `8.4.0`, `CURLOPT_PREREQFUNCTION` installs a callable that runs after the
connection is established and before the request is sent. Return
`CURL_PREREQFUNC_OK` to continue or `CURL_PREREQFUNC_ABORT` to cancel.

### Debug callback

Since `8.4.0`, `CURLOPT_DEBUGFUNCTION` receives:

1. the `CurlHandle`;
2. a `CURLINFO_*` debug-message type; and
3. the message string.

It cannot be combined with `CURLINFO_HEADER_OUT`, because both use the same
underlying cURL facility.

## Persistent shares, upload sizes, and redirects

Since `8.5.0`, cURL share handles can persist across PHP requests for safe
connection reuse.

Use `CURLOPT_INFILESIZE_LARGE` instead of `CURLOPT_INFILESIZE` when the older
option is constrained to a signed 32-bit size even on a 64-bit system.

`CURLOPT_FOLLOWLOCATION` accepts explicit redirect modes:

- `CURLFOLLOW_OBEYCODE` for stricter redirect-code handling;
- `CURLFOLLOW_FIRSTONLY` to stop after the first redirect; and
- `CURLFOLLOW_ALL`, equivalent to `true`.

```php
curl_setopt($handle, CURLOPT_FOLLOWLOCATION, CURLFOLLOW_FIRSTONLY);
```

## OpenSSL keys and password hashing

Since `8.4.0`, the OpenSSL extension supports x25519, ed25519, x448, and ed448
keys in key creation, key details, signing, and verification.

`PASSWORD_ARGON2` hashing is available when PHP is linked to OpenSSL 3.2 or
later in a non-thread-safe build (`8.4.0`). Detect the actual build capability
before selecting the algorithm.

The `key_length` argument to `openssl_pkey_derive()` is deprecated in
`8.5-migration`: it is either ignored or truncates the derived key, which can
be unsafe. Derive the complete key and perform an appropriate, explicit key
derivation or expansion step when a different length is needed.

## PCRE syntax and compatibility

The bundled PCRE2 10.44 behavior in `8.4-migration` requires pattern audits:

- `{,3}` is parsed as a quantifier rather than literal text.
- Some character classes have changed meaning in UCP mode.

Features available in `8.4.0` include:

- variable-length lookbehind assertions;
- spaces between braces in Perl-compatible items;
- named capture names up to 128 characters instead of 32; and
- the `r` or `(?r)` modifier, which with `i` prevents ASCII and non-ASCII
  characters from mixing in caseless matches.

In `8.5-migration`, PCRE is built without
`PCRE2_EXTRA_ALLOW_LOOKAROUND_BSK`; revise patterns that depend on that
extension.

## Stream objects and locking

`stream_bucket_make_writeable()` and `stream_bucket_new()` return
`StreamBucket`, not `stdClass`, in `8.4-migration`. Update type assertions and
annotations.

Since `8.5.0`, `flock()` supports zlib streams instead of always failing to
lock them.

## Readline history

Since `8.4.0`, the `PHP_HISTFILE` environment variable changes the history
path used in place of the default `.php_history`.

```sh
PHP_HISTFILE=/path/to/.php_history
```

## HTTP response headers

The local `$http_response_header` variable is deprecated in `8.5-migration`.
Use `http_get_last_response_headers()` to retrieve the most recent wrapper
response headers.

## Sendmail failures

Since `8.5.0`, `mail()` using the sendmail transport reports the actual
sendmail error, emits a warning, and returns `false` if sending fails or the
process exits unexpectedly. Check the return value and capture diagnostics;
do not assume process launch means delivery succeeded.
