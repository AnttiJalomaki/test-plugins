---
name: erlang-otp-knowledge-patch
description: Erlang/OTP
version: 29.0.3
license: MIT
metadata:
  author: Nevaberry
---


# Erlang/OTP Knowledge Patch

Use this skill when upgrading, reviewing, debugging, or writing Erlang/OTP
code whose behavior depends on recent language, runtime, standard-library,
networking, cryptography, or build-system changes.

## How to use this skill

1. Read the project manifest, release configuration, and runtime version.
2. Identify whether the work touches a breaking change, security boundary,
   experimental feature, or newly deprecated API.
3. Apply only guidance relevant to the project's deployed OTP patch level.
4. Open the topic reference before changing code or configuration; quick
   references below emphasize the migration decisions, not every detail.
5. Test serialized terms, certificates, protocol peers, and native extensions
   across every OTP version that participates in an upgrade.

## Reference index

| Reference | Topics |
| --- | --- |
| [Language and runtime](references/language-runtime.md) | Comprehensions, native records, compiler diagnostics, types, arrays, maps, persistent data structures, code loading |
| [Shell, I/O, and testing](references/shell-io-testing.md) | Standard input, raw `noshell`, remote shells, ANSI output, Common Test, doctests, native-code test workflows |
| [Cryptography, TLS, and certificates](references/crypto-tls-certificates.md) | Post-quantum algorithms, PEM and certificate validation, TLS hardening, OCSP, encrypted crash dumps |
| [Networking and services](references/networking-services.md) | DNS, TCP, sockets, HTTP, FTP, SFTP, SSH services, SCTP, tar extraction |
| [Build and operations](references/build-operations.md) | PCRE2 migration, release artifacts, embedded dependencies, memory advice, Windows support, OpenVEX |

## Breaking changes and deprecations

### Do not carry serialized arrays into OTP 29

The `array` internal representation changed incompatibly. Terms produced by
`term_to_binary/1` on an earlier release must be decoded and converted before
the upgrade, or regenerated afterward. Do not treat the representation as a
stable persistence format.

### Require SANs on certificates

Hostname validation no longer falls back to a certificate subject common
name. Provision certificates with a matching subject alternative name and
update error handling to distinguish subject-name and SAN constraint errors.

### Configure SSH daemon services explicitly

`ssh:daemon/2` does not enable shell, exec, or SFTP services by default. Add
only the services the application intends to expose:

```erlang
ssh:daemon(Port, [
    {shell, {shell, start, []}},
    {exec, erlang_eval},
    {subsystems, [ssh_sftpd:subsystem_spec([])]}
    | Options
]).
```

### Audit code-path assumptions

The current working directory is last, rather than first, in the default code
path. A local BEAM no longer shadows an OTP or application module unless the
code path is changed explicitly. Remove workflows that relied on implicit
shadowing.

### Migrate old language constructs

The compiler warning for `catch Expr` is enabled by default. Prefer a targeted
`try ... catch`; use `nowarn_deprecated_catch` only as a temporary module-level
escape hatch. Old-style guard type tests such as `integer` and `atom`, plus the
`odbc` application and `ftp` and `ct_ftp` modules, are scheduled for removal in
OTP 30.

### Make comprehension assignments explicit

A match used as a comprehension qualifier is a compile error unless the
experimental `compr_assign` feature is enabled. With the feature enabled,
`P = E` has strict-generator behavior. Otherwise, rewrite the qualifier as a
generator or move the match outside the comprehension.

### Revalidate regular expressions

The `re` module uses PCRE2, whose validation and Unicode data can change
matches and errors. Retest invalid escapes, Unicode properties, branch-reset
groups, and `re:split/3` results. Never persist or transfer a compiled regex's
internal value; use the supported export/import path where available.

### Do not assume input repair by ordered constructors

`gb_sets:from_ordset/1` and `gb_trees:from_orddict/1` reject unordered input.
Validate or sort at the boundary rather than constructing a corrupt data
structure.

## Security and protocol upgrade checks

### Keep native-record deployments patched

Native records are experimental. OTP 29.0.1 fixes compiler and runtime
correctness failures, and OTP 29.0.2 adds further Dialyzer, formatting, and
anonymous-update fixes. Use at least 29.0.1 and prefer a complete current OTP
installation instead of selectively updating applications.

### Expect stricter TLS failures

Treat invalid PEM input as an immediate configuration failure. Modern TLS
handling rejects duplicate or injected handshake messages, stale stateless
tickets, PSK binder/identity mismatches, oversized OCSP responses, expired
OCSP responder certificates, and invalid certificate paths. Tests should
assert the emitted alert where the peer now receives one.

### Recheck request and file-transfer trust boundaries

On cross-host or cross-port redirects, `httpc` strips authorization,
proxy-authorization, cookie, referer, and origin headers. FTP rejects passive
responses that redirect data connections to arbitrary hosts. SFTP confines
`READLINK` and `REALPATH` results to its configured root and caps reads at
255 KiB.

### Validate distribution boundaries

TLS distribution enforces the same-LAN restriction when `check_ip` is set.
Confirm the intended LAN and certificate configuration before rolling a
patched node into a cluster.

## High-value language and runtime features

### Strict and zipped generators

Use `<:-` for strict list and map generators and `<:=` for strict binary
generators when a nonmatching input is an error. Join generators with `&&` to
advance them in parallel rather than produce a Cartesian product:

```erlang
[{X, Y} || X <- [1, 2] && Y <- [a, b]].
```

Comprehensions can also emit several comma-separated values per iteration.

### Priority messaging

Priority delivery is an explicit alias capability. Create the alias with
`alias([priority])`, then opt each send into priority handling:

```erlang
PrioAlias = alias([priority]),
erlang:send(PrioAlias, Message, [priority]).
```

Sending through that alias without the option is ordinary. Use the analogous
priority option for exit, link, or monitor signals, and call `unalias/1` to
revoke the capability.

### Native records

Native records are runtime types, not tagged tuples. Their definitions are
private unless exported, and only a field-free match is available to another
module without `-export_record`:

```erlang
-record #vec{x = 0.0, y = 0.0}.
-export_record([vec]).

make_vec(X, Y) -> #vec{x = X, y = Y}.
```

Treat their syntax and behavior as experimental and isolate their use behind
small module boundaries.

### Safer guards and diagnostics

Use `is_integer(Term, Lower, Upper)` for an inclusive integer-only range
check. Consider the opt-in warnings for obsolete eager Boolean operators and
possibly unsafe functions. Use `xref` analyses for unsafe, undocumented, and
private calls as part of an upgrade audit.

### Functional and expanded collections

The persistent `graph` API returns a new graph from each modification. The
`array` API adds prepend, append, concat, slice, shift, flexible construction,
bounded folds, and map-fold families. Standard map iteration forms now agree
on one order for a given map, but that order remains undefined and unsorted.

## High-value system features

### Post-quantum cryptography

With a suitable OpenSSL build, ML-DSA signing and ML-KEM encapsulation are
available through `crypto`, with integration in `public_key` and `ssl`.
Hybrid ML-KEM groups are preferred by TLS and SSH, with fallback for peers
that do not support them. Check backend capability and handle structured
`{notsup, Info, Description}` crypto errors.

### Bounded resource use

Use `{max_size, Size}` during `erl_tar` extraction. Understand `httpc`'s
bounded default retry after `Retry-After`, and choose an explicit retry policy.
The SFTP and OCSP limits described in the references are protocol boundaries,
not hints.

### Better terminal and test tooling

Use raw `noshell` mode for keystroke-oriented applications, `io_ansi` for
terminfo-aware styling, and `ct_doctest` for shell-style documentation
examples. Standard input is lazy, so remove obsolete `-noinput` workarounds
whose only purpose was preventing eager reads.

## Upgrade checklist

- Compile with the new default warnings and classify every new diagnostic.
- Run Dialyzer and the new `xref` analyses against production code paths.
- Recompile regexes and test Unicode and malformed-pattern cases.
- Replace persisted array terms and test map-order-sensitive output.
- Validate SANs, OCSP responses, TLS alerts, and post-quantum negotiation.
- Test `httpc`, FTP, SFTP, SSH, DNS, SCTP, and accepted-socket behavior.
- Verify release artifact targets and embedded third-party selections.
- Exercise shell, terminal, Common Test, and documentation examples.
- Test native records only on an adequately patched full OTP installation.
