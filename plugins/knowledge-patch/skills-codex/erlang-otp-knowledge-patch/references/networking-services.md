# Networking and Services

## DNS resolution

OTP 28.1 adds option-list variants for direct resolver calls:

- `inet_res:gethostbyname/4`
- `inet_res:getbyname/4`
- `inet_res:gethostbyaddr/3`

Use them to override resolver settings for one operation without modifying
shared resolver configuration.

The internal `inet_dns_tsig` and `inet_res` modules now verify the correct
TSIG timestamp. Their two undocumented DNS error atoms were corrected to the
RFC names `notauth` and `notzone`; update code that matches the former,
incorrect atoms.

## TCP and socket behavior

Starting in OTP 28.3, `gen_tcp` supports `TCP_KEEPCNT`, `TCP_KEEPIDLE`, and
`TCP_KEEPINTVL`. `TCP_USER_TIMEOUT` is supported by both `gen_tcp` and
`socket`. Use these controls when the application's failure-detection policy
must differ from operating-system defaults.

OTP 29.0 adds socket support for `recvmmsg()` and `sendmmsg()`, allowing
multiple messages to be received or sent in one call.

In OTP 29.0.1, an IPv6 SCTP socket produced by peeloff correctly inherits its
parent socket's options.

OTP 29.0.2 makes accepted `gen_tcp_socket` connections inherit options in the
same way as classic `gen_tcp`. Remove backend-specific workarounds only after
testing the deployed patch level.

## HTTP redirects and retries

After receiving `Retry-After`, `httpc:request/4,5` in OTP 28.4 retries once by
default, then returns the error response instead of retrying indefinitely.
The HTTP option `{autoretry, timeout()}` controls the behavior. Applications
that require unbounded retries must configure it or implement their own retry
policy.

```erlang
HttpOptions = [{autoretry, RetryTimeout}].
```

From OTP 29.0.2, a redirect whose host or port changes causes `httpc` to strip
these sensitive headers:

- `authorization`
- `proxy-authorization`
- `cookie`
- `referer`
- `origin`

A client that deliberately sends credentials across that boundary must make
the new target authorization explicit.

## FTP and SFTP confinement

OTP 29.0 deprecates the `ftp` and `ct_ftp` modules and schedules them for
removal in OTP 30.

In OTP 29.0.2, FTP clients reject passive-mode responses that point the data
connection at an arbitrary host. The SFTP server also confines `READLINK`:
results no longer expose host paths outside the configured server root, and
paths remain relative to that root.

OTP 29.0.3 additionally prevents `REALPATH` requests containing `..` from
revealing whether paths outside the root exist. It caps SFTP reads at 255 KiB;
clients must divide larger reads.

## SSH services and compatibility

`ssh:daemon/2` in OTP 29.0 no longer turns on shell, exec, or SFTP services by
default. Opt in to each required service:

```erlang
ssh:daemon(Port, [
    {shell, {shell, start, []}},
    {exec, erlang_eval},
    {subsystems, [ssh_sftpd:subsystem_spec([])]}
    | Options
]).
```

This is a behavioral change for existing daemons: test every expected service
after upgrade and avoid enabling unused capabilities.

OTP 29.0.3 removes the obsolete SHA-1 authentication workaround for OpenSSH
7.x. Retest any legacy peer that depended on that compatibility path.

For post-quantum SSH negotiation and its preferred default, see
[Cryptography, TLS, and Certificates](crypto-tls-certificates.md).

## TLS distribution

OTP 29.0.2 applies the same-LAN restriction to Erlang distribution over TLS
when `check_ip` is enabled. Connections admitted by older patch levels may be
rejected. Verify the desired LAN boundary and certificates during a rolling
upgrade.

## SNMP error responses

OTP 29.0.1 fixes `snmpm_usm:generate_outgoing_msg/5` so it no longer crashes
with `badmatch` while constructing an error response for an unknown user or
engine ID.

## Safe tar extraction

OTP 29.0 adds the `erl_tar` extraction option `{max_size, Size}` to cap total
extracted data and protect the destination from being filled. Symlink
validation now accepts safe relative targets such as `dir/link -> ../file`
that older releases rejected.
