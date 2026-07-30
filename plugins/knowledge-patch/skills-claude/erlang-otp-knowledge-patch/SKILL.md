---
name: erlang-otp-knowledge-patch
description: Erlang/OTP
version: 29.0.3
license: MIT
metadata:
  author: Nevaberry
---


# Erlang/OTP Knowledge Patch

Use this skill when upgrading Erlang/OTP, reviewing code that uses recent
language or runtime behavior, or diagnosing compatibility changes in the
compiler, standard applications, networking, cryptography, and build system.

## Reference index

| Reference | Topics |
| --- | --- |
| [language-and-compiler.md](references/language-and-compiler.md) | Comprehensions, native records, guards, literals, types, warnings, abstract forms, and compiler fixes |
| [runtime-and-data.md](references/runtime-and-data.md) | Priority messages, process memory, shell I/O, arrays, maps, trees, graphs, regexes, terminal output, and persistent terms |
| [crypto-tls-and-ssh.md](references/crypto-tls-and-ssh.md) | Post-quantum algorithms, certificates, TLS hardening, SSH defaults, SFTP confinement, and crypto errors |
| [networking-and-protocols.md](references/networking-and-protocols.md) | DNS, TCP, sockets, HTTP, FTP, SCTP, SNMP, and TLS distribution |
| [tooling-build-and-release.md](references/tooling-build-and-release.md) | `xref`, Common Test, doctests, tar extraction, native code, release artifacts, third-party libraries, VEX, and platforms |

## Start with breaking changes and deprecations

### Prepare for removals

- Replace old-style guard tests such as `integer` and `atom`; they are
  deprecated and scheduled for removal in OTP 30.
- Plan replacements for the deprecated `odbc` application and the `ftp` and
  `ct_ftp` modules, which have the same removal schedule.
- Replace old-style `catch Expr` with targeted `try ... catch` logic. The
  compiler warning is enabled by default.
- Audit eager `and` and `or` with `warn_obsolete_bool_op`; use `andalso`,
  `orelse`, or guard separators where appropriate.

### Restore SSH daemon services explicitly

`ssh:daemon/2` does not enable shell, exec, or SFTP services by default.
Configure every service the application intends to expose:

```erlang
ssh:daemon(Port, [
    {shell, {shell, start, []}},
    {exec, erlang_eval},
    {subsystems, [ssh_sftpd:subsystem_spec([])]}
    | Options
]).
```

Do not assume an existing daemon configuration still exposes the same
surface after an upgrade.

### Treat serialized arrays as release-bound

The `array` internal representation changed. Do not carry array terms created
with `term_to_binary/1` on an earlier installation into OTP 29 unchanged.
Decode and migrate them using a release-neutral representation.

### Recheck code-path assumptions

The current working directory is last, not first, in the default code path.
A local BEAM file no longer shadows an OTP or application module unless the
path is changed explicitly. Remove workflows that depended on implicit local
shadowing.

### Supply ordered input to ordered constructors

`gb_sets:from_ordset/1` and `gb_trees:from_orddict/1` reject unordered input
instead of constructing an invalid data structure. Sort or validate input
before calling them.

### Use SAN certificates

Hostname validation no longer falls back to the certificate subject common
name. Certificates need a matching subject alternative name. Error matching
must distinguish subject-name and subject-alternative-name constraint
failures.

## Security-sensitive upgrade checks

### Update complete installations

Avoid mixing individual application patches unless their documented
dependencies are also installed:

- PKICMP support in `public_key` depends on the OpenSSL-backed `crypto`
  version that introduced it.
- Stricter TLS duplicate-message handling requires corresponding `crypto`
  and `public_key` versions.
- `ssl-11.7.1` requires `public_key-1.21.1`.
- Native-record fixes span compiler, ERTS, Dialyzer, and formatting code;
  update the full installation.

### Expect stricter TLS and certificate failures

- Missing or invalid configured PEM files fail earlier during server setup.
- Duplicate TLS 1.3 handshake messages, a second HelloRetryRequest, injected
  plaintext-window application data, and invalid ticket or PSK state are
  rejected.
- Expired OCSP responder certificates and invalid certificate-path basic
  constraints are rejected.
- OCSP responses larger than 100 KB are rejected before ASN.1 decoding.
- Malformed clients can receive an alert where older servers silently closed
  the connection.

### Revalidate redirect and file-service boundaries

- `httpc` strips authorization, proxy-authorization, cookie, referer, and
  origin headers when a redirect changes host or port.
- FTP passive mode rejects responses that redirect the data connection to an
  arbitrary host.
- SFTP `READLINK` and `REALPATH` remain confined to the configured root, and
  reads are limited to 255 KiB per request.
- TLS distribution applies the same-LAN restriction when `check_ip` is
  enabled.

## Language quick reference

### Choose strict or zipped generators deliberately

Strict generators raise on a nonmatching input:

```erlang
[X || {ok, X} <:- [{ok, 1}, error, {ok, 3}]].
```

Use `<:-` for list and map generators and `<:=` for binary generators. Keep
the traditional operators when filtering nonmatches is intentional.

Zip generators with `&&` to consume them in parallel:

```erlang
[{X, Y} || X <- [1, 2] && Y <- [a, b]].
%% [{1,a},{2,b}]
```

Zipping is not a Cartesian product and supports list, binary, and map
generators.

### Gate experimental comprehension assignment

A match used as a qualifier is a compile error by default. Enable the
`compr_assign` feature before using assignment qualifiers:

```erlang
-feature(compr_assign, enable).
hashes(Items) ->
    [H || Item <- Items, H = erlang:phash2(Item), H rem 10 =:= 0].
```

The assignment acts like a strict single-element generator. Comprehensions
can also emit multiple comma-separated values per iteration.

### Treat native records as experimental

Declare a native record with `-record #name{...}.`. It is a runtime type, not
a tagged tuple:

```erlang
-record #vec{x = 0.0, y = 0.0}.
-export_record([vec]).

make_vec(X, Y) -> #vec{x = X, y = Y}.
```

Definitions are private by default. Other modules may make field-free
matches, but construction and field-aware matching require
`-export_record/1`. Use a current patch release because early compiler,
runtime, analysis, and formatting defects were corrected.

### Prefer the bounded integer guard

`is_integer(Term, LowerBound, UpperBound)` checks that all arguments are
integers and that the term is within the inclusive range. It avoids range
guards that accidentally accept floats.

## Runtime quick reference

### Send priority messages through a capability

Create a priority-capable alias and use the `priority` send option:

```erlang
PrioAlias = alias([priority]),
erlang:send(PrioAlias, Message, [priority]).
```

Sending through the alias without the option is ordinary. `unalias/1`
revokes the capability. Priority exit, link, and monitor signaling are
available through their corresponding priority options.

### Hibernate without discarding the stack

`erlang:hibernate/0` minimizes the calling process while it waits for its next
message and preserves the call stack. This differs from
`erlang:hibernate/3`.

### Insert persistent terms idempotently

`persistent_term:put_new/2` returns quickly when the key already holds the
same value and raises `badarg` when it holds a different value:

```erlang
persistent_term:put_new(config, Config).
```

### Use functional graphs for persistent updates

The `graph` module is a persistent counterpart to `digraph`; each modifying
operation returns a new graph and leaves earlier values usable.

## Operational checklist

1. Compile with default warnings plus `warn_obsolete_bool_op` and, where
   appropriate, `warn_possibly_unsafe_function`.
2. Run `xref` unsafe, undocumented, and private-call analyses; handle error
   results for BEAM files without debug information.
3. Retest regular expressions under PCRE2 and transfer compiled patterns only
   through the supported export/import mechanism.
4. Exercise certificate, TLS, SSH, redirect, FTP, SFTP, and distribution
   boundaries with negative tests.
5. Verify TCP keepalive, timeout, accepted-socket inheritance, and batched
   socket operations on each supported operating system.
6. Check release packaging, embedded third-party selection, native-code
   loading, crash-dump requirements, and Windows architecture support.
7. Run Common Test and documentation examples; update output snapshots that
   depend on map ordering or protocol alerts.

