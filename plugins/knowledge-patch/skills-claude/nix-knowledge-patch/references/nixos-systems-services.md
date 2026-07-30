# NixOS Systems and Services

Use this reference when upgrading NixOS configurations, rebuilding or switching
systems, migrating boot and network behavior, or adapting service modules.

## Rebuilds, images, and configuration entry points

### Opting in to `nixos-rebuild-ng`

In nixos-25.05, set `system.rebuild.enableNg = true` to replace the original
`nixos-rebuild` with the Python rewrite. Installing `nixos-rebuild-ng` in
`environment.systemPackages` instead made it available side by side. Later
releases make the rewrite the default.

### Building system images with `nixos-rebuild`

Since nixos-25.05, `nixos-rebuild build-image` builds the platform-specific
disk image defined by the configuration. It works with both rebuild
implementations.

### Flake-aware `nixos-option`

Since nixos-25.05, `nixos-option` supports flake configurations, descent into
`attrsOf` and `listOf` submodules, and `--show-trace`.

### Channel tarballs as lockable flake inputs

Since nixos-25.05, the channel server implements the Lockable HTTP Tarball
Protocol. A channel `nixexprs.tar.xz` can be used directly as a flake input:

```nix
inputs.nixpkgs.url =
  "https://channels.nixos.org/nixos-25.05/nixexprs.tar.xz";
```

### Customizable image output names

In nixos-25.05, default filenames from `system.build` image builders changed.
Use `image.baseName`, `image.extension`, and `image.fileName`; `image.filePath`
exposes the evaluation-time path relative to the derivation output.

### `nixos-rebuild-ng` is the default

In nixos-25.11, the Python rewrite became the default. Setting
`system.rebuild.enableNg = false` was only a temporary opt-out.

### Rust-only system switching

In nixos-25.11, the Perl `switch-to-configuration` was removed. Every switchable
system uses the Rust implementation; remove `system.switch.enableNg`.

### Channel-free `system.nix` entry points

Since nixos-26.05, `/etc/nixos/system.nix` may evaluate directly to one system
derivation or an attribute set selected by `nixos-rebuild --attr`. `--file`
selects another file or directory, and `--attr` also considers `system.nix` in
the current directory. This enables pinned, channel-free entry points.

### Switch inhibitors and unit activation

In nixos-26.05, switch inhibitors can reject a generation when configured
comparison strings differ; `NIXOS_NO_CHECK=1` forces a switch. A unit reloads
when its only change is `ExecReload=`, while removing `ExecReload=` does
nothing. Activation-script-driven unit reloads and restarts are deprecated.

## State versions and core module contracts

### Validated system state versions

Since nixos-25.05, `system.stateVersion` must use the `"YY.MM"` form matching a
NixOS release.

### Overriding module-provided file systems

In nixos-25.05, NixOS-provided `fileSystems` definitions use `lib.mkDefault`,
allowing complete replacement. Overriding only `fsType` or `options` can lose
the required `device`; restate it explicitly.

### State-gated PostgreSQL 17 default

With `system.stateVersion = "25.11"` or newer in nixos-25.11, new systems
default to PostgreSQL 17. Older state versions retain their selected default.

### Stricter core option types

In nixos-26.05:

- `services.openssh.settings.AcceptEnv` must be a list, not a string.
- Every `fileSystems.<name>.fsType` must be explicit.
- Unknown `services.xserver.videoDriver(s)` values fail evaluation.

```nix
services.openssh.settings.AcceptEnv = [ "LANG" "LC_*" ];
fileSystems."/".fsType = "ext4";
```

### Structured system configuration migrations

In nixos-26.05, migrate free-form settings:

- `systemd.coredump.extraConfig` to
  `systemd.coredump.settings.Coredump`;
- `systemd.sleep.extraConfig` to `systemd.sleep.settings.Sleep`;
- `services.pdns-recursor.yaml-settings` to
  `services.pdns-recursor.settings`.

Resolved and Dovecot similarly expose RFC 42-style settings.

## Boot, initrd, containers, and system facilities

### Bashless systemd initrd initialization

In nixos-25.11, `system.nixos-init.enable = true` enables the Rust `nixos-init`,
allowing a systemd initrd without an interpreter.

### Imperative containers require an explicit boot opt-in

In nixos-25.11, `boot.enableContainers` is automatic only when declarative
`containers` exist. Hosts managed imperatively with `nixos-container` must set:

```nix
boot.enableContainers = true;
```

### Nix store mount option migration

In nixos-25.11, `boot.readOnlyNixStore` was removed. Configure `/nix/store`
bind-mount behavior with `boot.nixStoreMountOpts`.

### Systemd is the default stage 1

In nixos-26.05, the initrd uses systemd by default. Scripted stage 1 is
deprecated and can be retained temporarily with
`boot.initrd.systemd.enable = false`. LUKS roots should name their
`/dev/mapper/...` device, `/dev/root` must become a stable path, and complex
LVM-on-LUKS may need an infinite systemd device timeout.

```nix
boot.initrd.luks.devices.cryptroot.device = "/dev/disk/by-uuid/...";
fileSystems."/".device = "/dev/mapper/cryptroot";
```

### Container-backed NixOS tests

Since nixos-26.05, the integration-test driver can use `systemd-nspawn`
containers rather than QEMU, including on builders without KVM and for tests
that bind-mount host devices such as GPUs.

### Removed system facilities

In nixos-26.05, `profiles/hardened`, `linux_hardened`, `linux-rt`, ReiserFS, and
eCryptfs support were removed. Systemd no longer starts units installed by
`nix-env -i`. Replace `post-resume.target` ordering with `sleep.target` and
`ExecStop=`.

## Networking, firewall, and wireless

### Services must request online networking explicitly

Since nixos-25.05, `multi-user.target` is not ordered after
`network-online.target`. A service that requires connectivity must declare both
the dependency and ordering:

```nix
systemd.services."<name>" = {
  wants = [ "network-online.target" ];
  after = [ "network-online.target" ];
};
```

### NAT external-address filtering

In nixos-25.05, when `networking.nat.externalIP` or `externalIPv6` is set,
`forwardPorts` handles only packets addressed to that configured external
address.

### Networkd-backed WireGuard

In nixos-25.05, `networking.wireguard` can use a networkd backend. It is selected
when `networking.useNetworkd` is enabled or explicitly through
`networking.wireguard.useNetworkd`; semantics differ from the scripted backend.

### FirewallD and selectable firewall backends

Since nixos-25.11, `services.firewalld` runs FirewallD, and
`networking.firewall.backend` selects the backend for existing firewall
configuration.

### Explicit NetworkManager VPN plugins

In nixos-25.11, NetworkManager has no default VPN plugin set. List every needed
plugin in `networking.networkmanager.plugins`.

### Hardened wireless configuration

In nixos-26.05, `wpa_supplicant` runs unprivileged. Generated and imperative
configuration lives below `/etc/wpa_supplicant`, and credentials must be
readable by its user. Remove `networking.wireless.userControlled.group`;
`.userControlled.enable` becomes `.userControlled`.

NetworkManager relies on `networking.wireless`, so remove an explicit
`networking.wireless.enable = false`. `networking.wireless.enableHardening` is
the compatibility escape hatch. `iw` and `wirelesstools` are no longer
implicitly installed.

### Network and resolver ordering

In nixos-26.05, the scripted backend removes `network-setup.service`; addresses,
routes, and gateways apply asynchronously as devices appear. Name-server setup
runs in `network-local-commands.service`. `networking.resolvconf.enable` always
defaults to `true`; disable it explicitly when managing `/etc/resolv.conf`.

### Firewall refusal logging is opt-in

In nixos-26.05, `networking.firewall.logRefusedConnections` defaults to `false`
to avoid flooding the kernel log. Enable it explicitly when refused-packet logs
are required.

## System services and structured settings

### Hardened and correctly escaped `earlyoom`

In nixos-25.05, the module uses upstream's hardened systemd unit. A `killHook`
that needs home or filesystem access can require a `ProtectSystem` override.
Every `extraArgs` item is independently shell-escaped rather than word-split.

```nix
services.earlyoom.extraArgs = [ "--prefer" "spaced pat" ];
```

### BorgBackup hook arguments are arrays

In nixos-25.05, `services.borgbackup.jobs.*.extraArgs` and the other
`extra*Args` values are Bash arrays. Hooks must append array elements:

```sh
extraCreateArgs+=("--exclude" "/some/path")
```

### AppArmor policy state

In nixos-25.05, `security.apparmor.policies.<name>.enable` and `.enforce` were
removed. Use the `state` tristate option.

### Structured rsyncd settings

In nixos-25.05, `services.rsyncd.settings` accepts only `sections` and
`globalSection`. Move named sections below `settings.sections` and former
`settings.global` values below `settings.globalSection`.

### Structured NixOS settings migrations

In nixos-25.11:

- `services.dwm-status.extraConfig` becomes `.settings`, with `order` nested.
- `services.traccar.settings.loggerConsole` becomes
  `services.traccar.settings.logger.console`.
- `services.logind.extraConfig` becomes `services.logind.settings.Login`.
- `systemd.extraConfig` and `boot.initrd.systemd.extraConfig` move to the
  corresponding `systemd.settings.Manager`.
- Watchdogs use `RuntimeWatchdogSec`, `WatchdogDevice`, `RebootWatchdogSec`, and
  `KExecWatchdogSec` under `systemd.settings.Manager`.
- `systemd.enableCgroupAccounting` becomes the individual `*Accounting`
  manager settings.

### Postfix TLS certificate settings

In nixos-25.11, `services.postfix.sslCert` and `sslKey` were removed. Use
`services.postfix.settings.main.smtpd_tls_chain_files` for server chains and
`smtp_tls_chain_files` for client chains.

### GNOME SSH agent split

In nixos-25.11, `gnome-keyring` no longer supplies an SSH agent. Use
`services.gnome.gcr-ssh-agent.enable` from `gcr_4`; it defaults to the value of
`services.gnome.gnome-keyring.enable` for migration compatibility.

### Nettools is no longer installed by default

In nixos-25.11, `ifconfig`, `arp`, `mii-tool`, `netstat`, and `route` are absent
from default installations. Prefer `iproute2` and `ethtool`, or install
`nettools` explicitly.

### Declarative user lingering is opt-in

In nixos-25.11, `users.users.<name>.linger` defaults to `null`, preserving the
existing loginctl state rather than disabling lingering.
`users.manageLingering` disables NixOS lingering management globally.

### `dbus-broker` is the default

In nixos-26.05, `services.dbus.implementation` defaults to `dbus-broker`.
Changing implementations is a switch inhibitor and requires a reboot. Pin
`"dbus"` to keep the reference daemon.

### OpenSSH module changes

In nixos-26.05, `services.openssh.generateHostKeys = true` can generate keys
while the daemon is disabled. Set `enableRecommendedAlgorithms = false` to
disable the curated algorithm set. Replace removed `services.openssh.banner`
with `services.openssh.settings.Banner`.

### NVIDIA branch and module configuration

In nixos-26.05, `hardware.nvidia.branch` chooses a driver branch unless
`hardware.nvidia.package` overrides it. `hardware.nvidia.moduleParams` writes
modprobe settings. Nixpkgs exposes `production`, `new_feature`, and `beta`;
proprietary modules moved to `nvidia_x11.mod`. Maxwell-or-older GPUs must pin
the retained 580 legacy driver.

### Explicit service access control

In nixos-26.05, VSFTPD no longer creates a PAM service automatically:
`localUsers` needs an explicitly enabled PAM service or a virtual-user database.
Cgit must explicitly choose `gitHttpBackend.checkExportOkFiles`; do not inherit
an export-all backend that bypasses cgit access controls.

### Locally built Kubernetes CoreDNS

In nixos-26.05, `services.kubernetes.addons.dns.coredns` becomes `corednsImage`
and takes an image package, not attributes. The default is locally built from
`pkgs.coredns` with `dockerTools.buildImage`; use a `dockerTools.pullImage`
derivation to retain an upstream image.

## Certificates and secrets

### ACME unit dependency changes

In nixos-25.11, services requiring a syntactically valid certificate should
depend on `acme-{certname}.service`. Initial self-signed certificates are always
generated, `security.acme.preliminarySelfsigned` is removed, and dependencies
on `acme-finished-{certname}.target` move to
`acme-order-renew-{certname}.service`.

### Lifetime-aware ACME renewal

In nixos-26.05, when `security.acme.defaults.validMinDays` is unset,
certificates valid for at least ten days renew after two thirds of their
lifetime; shorter-lived certificates renew halfway through.

### Secret-file migrations

In nixos-26.05, `services.oauth2-proxy.clientSecret` and `cookie.secret` become
`clientSecretFile` and `cookie.secretFile`. Grafana's
`settings.security.secret_key` has no default; retain or rotate the old value
deliberately and inject it through Grafana variable expansion rather than the
Nix store.

### Secure Yggdrasil keys

In nixos-26.05, removed `services.yggdrasil.configFile` and `persistentKeys`
become structured `settings`. Persistent private keys must be PKCS #8 PEM files
referenced by `settings.PrivateKeyPath`. A literal `PrivateKey` in `settings` is
rejected to prevent store disclosure.

## Databases and applications

### Nextcloud database and credential defaults

In nixos-25.05, `services.nextcloud.config.dbtype` has no SQLite default and
must be explicit. Secret files use systemd credentials, so `nextcloud-occ`
needs root or an existing `$CREDENTIALS_DIRECTORY`.

### PostgreSQL readiness target

In nixos-25.11, `postgresql.target` guarantees a writable database after
initialization and ensure scripts. `postgresql.service` guarantees only a
read-only connection.

### Forgejo dump retention default

In nixos-25.11, `services.forgejo.dump.age` defaults to `4w`; older dumps are
deleted unless retention is overridden.

### Nextcloud cache and major-version upgrades

In nixos-25.11, `services.nextcloud.configureRedis` defaults to `true`.
For `system.stateVersion >= "25.05"`, the implicit package is Nextcloud 32.
Upgrades from 30 or older must pin and pass through Nextcloud 31.

### State-gated service data migrations

With state version 26.05 in nixos-26.05, TaskChampion Sync Server uses
`DynamicUser` and `/var/lib/private/taskchampion-sync-server`; migrate the old
directory when opting in manually. Renamed `services.stalwart` has its own
`stateVersion`; user, group, data-directory, and tracer defaults change when it
reaches 26.05.

### Mattermost 11 requires PostgreSQL

In nixos-26.05, the Mattermost module defaults to version 11 and removes MySQL.
Remove the upstream 250-user limit only by overriding the selected package with
`removeUserLimit`, optionally with `removeFreeBadge`.

### Nextcloud 33 upgrade stepping

With `system.stateVersion >= "26.05"` in nixos-26.05, Nextcloud defaults to 33
and Nextcloud 31 is removed. Systems on 31 or older must explicitly pin and
upgrade through `pkgs.nextcloud32`.

### Immich vector-extension migration

In nixos-26.05, Immich removes `database.enableVectors` and
`database.enableVectorchord` and always uses VectorChord. Completely remove
pgvecto.rs from an existing database before upgrading.

## Module and package selection migrations

### Selecting the system Mesa package

Since nixos-25.05, `hardware.graphics.package` can choose the global Mesa
version without triggering a mass rebuild.

### Locale configuration

In nixos-25.05, prefer `i18n.extraLocales` for additional locales.
`i18n.supportedLocales` remains but is an implementation detail and warns when
required locales are omitted. Use `i18n.defaultCharset` and
`i18n.localeCharsets` for per-locale character sets.

### Renamed and replacement NixOS modules

In nixos-25.11, migrate:

- `programs.river` to `programs.river-classic`;
- `services.nixseparatedebuginfod` to `services.nixseparatedebuginfod2`;
- `services.dnscrypt-proxy2` to `services.dnscrypt-proxy`;
- `services.pds` to `services.bluesky-pds`;
- removed `virtualisation.lxd` to `virtualisation.incus`.
