# NixOS System and Service Migrations

## Rebuild, inspection, and image workflows

### Rebuild implementation

The Python `nixos-rebuild-ng` rewrite can be selected in nixos-25.05 with:

```nix
system.rebuild.enableNg = true;
```

Installing `nixos-rebuild-ng` in `environment.systemPackages` instead keeps it
alongside the original command.

It becomes the default in nixos-25.11. `system.rebuild.enableNg = false` is a
temporary opt-out expected to disappear. The Perl
`switch-to-configuration` implementation is removed; remove the obsolete
`system.switch.enableNg` option because all switchable systems use Rust.

### Images and option inspection

`nixos-rebuild build-image` builds the platform-specific disk image declared
by a system configuration (since nixos-25.05), and is supported by both rebuild
implementations.

Image builder output names changed in nixos-25.05. Customize them with
`image.baseName`, `image.extension`, and `image.fileName`; `image.filePath`
exposes the evaluation-time path relative to the derivation output.

`nixos-option` supports flakes, descent into `attrsOf` and `listOf` submodules,
and `--show-trace` as of nixos-25.05.

### Channel-free entry points

`/etc/nixos/system.nix` can evaluate directly to a NixOS system derivation or
an attribute set selected with `nixos-rebuild --attr` (since nixos-26.05).
`--file` selects another file or directory, and `--attr` also considers
`system.nix` in the current directory. This enables a pinned entry point
without `nix-channel`.

```nix
let
  nixpkgs = builtins.fetchTarball {
    url = "https://github.com/NixOS/nixpkgs/archive/c217913993d6.tar.gz";
    sha256 = "026mprs324330pfazlgbw987qmsa8ligglarvqbcxzig2kgw0lqg";
  };
in
import "${nixpkgs}/nixos" { configuration = ./configuration.nix; }
```

### Container-backed tests

The integration test driver can run machines as `systemd-nspawn` containers
instead of QEMU (since nixos-26.05). This works on VM builders without KVM and
supports tests that bind-mount host devices such as GPUs.

## State versions and option evaluation

`system.stateVersion` is validated as a NixOS release string in `YY.MM` form
(since nixos-25.05).

Core types become stricter in nixos-26.05:

- `services.openssh.settings.AcceptEnv` is a list, not a string.
- Every `fileSystems.<name>.fsType` is explicit.
- Unknown `services.xserver.videoDriver` or `videoDrivers` values fail
  evaluation rather than being ignored.

```nix
services.openssh.settings.AcceptEnv = [ "LANG" "LC_*" ];
fileSystems."/".fsType = "ext4";
```

NixOS-supplied `fileSystems` definitions use `lib.mkDefault` as of
nixos-25.05, permitting wholesale replacement. Overriding only an attribute
such as `fsType` or `options` can lose the required `device`; state it
explicitly when that occurs.

## Boot, initrd, switching, and storage

### Stage 1

Systemd is the default initrd implementation in nixos-26.05. Scripted stage 1
is deprecated for removal in 26.11 and can temporarily be retained with:

```nix
boot.initrd.systemd.enable = false;
```

For systemd stage 1:

- Name LUKS roots with their matching `/dev/mapper/...` device.
- Replace `/dev/root` with a stable device path.
- Complex LVM-on-LUKS roots may need an infinite systemd device timeout.

```nix
boot.initrd.luks.devices.cryptroot.device = "/dev/disk/by-uuid/...";
fileSystems."/".device = "/dev/mapper/cryptroot";
```

The Rust `nixos-init` added in nixos-25.11 allows a systemd initrd with no
interpreter:

```nix
system.nixos-init.enable = true;
```

### Nix store mount

`boot.readOnlyNixStore` was removed in nixos-25.11. Configure the `/nix/store`
bind mount through `boot.nixStoreMountOpts`.

### Switch behavior

Switch inhibitors in nixos-26.05 can reject a generation when configured
comparison strings differ. `NIXOS_NO_CHECK=1` is the force override.

`switch-to-configuration` reloads a unit when its only change is `ExecReload=`
and does nothing when `ExecReload=` is removed. Unit reloads or restarts
requested from activation scripts are deprecated; express activation through
unit changes.

### Removed facilities

Nixos-26.05 removes the `profiles/hardened`, `linux_hardened`, and `linux-rt`
choices, along with ReiserFS and eCryptfs support. Systemd no longer starts
units installed through `nix-env -i`. Replace `post-resume.target` ordering
with `sleep.target` and `ExecStop=`.

## System manager and structured settings

Nixos-25.11 moves free-form manager configuration to RFC 42-style attributes:

- `systemd.extraConfig` becomes `systemd.settings.Manager`.
- `boot.initrd.systemd.extraConfig` becomes
  `boot.initrd.systemd.settings.Manager`.
- Manager watchdog keys are `RuntimeWatchdogSec`, `WatchdogDevice`,
  `RebootWatchdogSec`, and `KExecWatchdogSec`.
- `systemd.enableCgroupAccounting` is replaced by the individual
  `*Accounting` manager settings.
- `services.logind.extraConfig` becomes `services.logind.settings.Login`.

The same release moves `services.dwm-status.extraConfig` to
`services.dwm-status.settings` with nested `order`, and moves
`services.traccar.settings.loggerConsole` to
`services.traccar.settings.logger.console`.

Nixos-26.05 continues the structured migration:

- `systemd.coredump.extraConfig` becomes
  `systemd.coredump.settings.Coredump`.
- `systemd.sleep.extraConfig` becomes `systemd.sleep.settings.Sleep`.
- `services.pdns-recursor.yaml-settings` becomes
  `services.pdns-recursor.settings`.
- Resolved and Dovecot likewise expose RFC 42-style settings.

```nix
systemd.coredump.settings.Coredump.Storage = "journal";
```

## Networking and name resolution

### Online ordering

`multi-user.target` is no longer ordered after `network-online.target` as of
nixos-25.05. A service that cannot start offline needs both:

```nix
systemd.services."<name>" = {
  wants = [ "network-online.target" ];
  after = [ "network-online.target" ];
};
```

### Firewall and NAT

With `networking.nat.externalIP` or `externalIPv6` set, `forwardPorts` matches
only packets whose destination is that configured external address (since
nixos-25.05).

Nixos-25.11 can run FirewallD through `services.firewalld` or select it as the
backend for `networking.firewall` with `networking.firewall.backend`.

`networking.firewall.logRefusedConnections` defaults to `false` in
nixos-26.05, avoiding kernel-ring-buffer floods. Enable it explicitly when
refused-packet logging is required.

### WireGuard and NetworkManager

`networking.wireguard` has an optional networkd backend in nixos-25.05. It is
selected automatically by `networking.useNetworkd`, or explicitly with
`networking.wireguard.useNetworkd`; review backend-specific option semantics.

NetworkManager has no default VPN plugin set in nixos-25.11. List every needed
plugin in `networking.networkmanager.plugins`.

### Wireless hardening

In nixos-26.05, `wpa_supplicant` runs unprivileged. Generated and imperative
configuration moves below `/etc/wpa_supplicant`, and referenced credentials
must be readable by that user.

- Remove `networking.wireless.userControlled.group`.
- Rename `.userControlled.enable` to `.userControlled`.
- Remove an explicit `networking.wireless.enable = false` when using
  NetworkManager; it now relies on `networking.wireless`.
- Use `networking.wireless.enableHardening` only as a compatibility escape.
- Add `iw` or `wirelesstools` explicitly if needed; neither is implicit.

### Scripted networking and resolver ordering

The scripted backend in nixos-26.05 has no `network-setup.service`. Addresses,
routes, and gateways are applied asynchronously as devices appear; name-server
setup runs in `network-local-commands.service`.

`networking.resolvconf.enable` always defaults to `true`. A configuration that
owns `/etc/resolv.conf` must set:

```nix
networking.resolvconf.enable = false;
```

## OpenSSH and access control

OpenSSH in nixos-26.05 can generate host keys while the daemon is disabled via
`services.openssh.generateHostKeys = true`. Set
`enableRecommendedAlgorithms = false` to disable the curated algorithm set.
`services.openssh.banner` is removed; use
`services.openssh.settings.Banner`.

VSFTPD no longer creates a PAM service automatically in nixos-26.05. Local
users require an explicitly enabled PAM service or a virtual-user database.

Cgit must explicitly choose `gitHttpBackend.checkExportOkFiles`; it no longer
inherits an export-all backend that bypasses cgit access controls.

## D-Bus, containers, and user sessions

NixOS defaults to `services.dbus.implementation = "dbus-broker"` in
nixos-26.05. Switching implementations is a switch inhibitor and requires a
reboot. Pin `"dbus"` to retain the reference daemon.

`boot.enableContainers` is automatically enabled only when declarative
`containers` exist as of nixos-25.11. Imperatively managed `nixos-container`
hosts must set:

```nix
boot.enableContainers = true;
```

`users.users.<name>.linger` defaults to `null` in nixos-25.11, leaving
existing `loginctl` state untouched. `users.manageLingering` can disable NixOS
management of lingering globally.

Legacy `ifconfig`, `arp`, `mii-tool`, `netstat`, and `route` are not installed
by default in nixos-25.11. Use `iproute2` and `ethtool`, or explicitly install
`nettools`.

## Databases and stateful applications

### PostgreSQL

With `system.stateVersion = "25.11"` or newer, new systems default to
PostgreSQL 17. Older state versions retain their selected version.

Depending on `postgresql.target` in nixos-25.11 guarantees a writable database
after initialization and ensure scripts. Depending on `postgresql.service`
guarantees only a read-only connection.

### Nextcloud

In nixos-25.05, `services.nextcloud.config.dbtype` has no SQLite default and
must be set. Secret files use systemd credentials, so `nextcloud-occ` needs
root privileges or an existing `CREDENTIALS_DIRECTORY`.

`services.nextcloud.configureRedis` defaults to `true` in nixos-25.11. With
state version at least 25.05, the implicit package is Nextcloud 32. Because
upgrades from 30 or earlier cannot skip major versions, pin and pass through
Nextcloud 31.

With state version at least 26.05, the default becomes Nextcloud 33 and
Nextcloud 31 is removed. An installation on 31 or older must pin and upgrade
through 32 first:

```nix
services.nextcloud.package = pkgs.nextcloud32;
```

### Mattermost, Immich, and service data

The Mattermost module defaults to version 11 and removes MySQL support in
nixos-26.05. Removing the upstream 250-user limit requires overriding the
selected package with `removeUserLimit`, optionally with `removeFreeBadge`.

The Immich module always uses VectorChord in nixos-26.05; the
`database.enableVectors` and `database.enableVectorchord` options are removed.
Completely remove an existing pgvecto.rs extension from the database before
upgrading.

At state version 26.05, TaskChampion Sync Server enables `DynamicUser` and
moves data to `/var/lib/private/taskchampion-sync-server`; migrate the old
directory when opting in manually. The renamed `services.stalwart` module has
its own `stateVersion` and changes user, group, data path, and tracer defaults
only when that version reaches 26.05.

## Certificates and secrets

### ACME

Nixos-25.11 always creates initial self-signed certificates and removes
`security.acme.preliminarySelfsigned`. A service needing syntactically valid
material should depend on `acme-{certname}.service`. Replace dependencies on
`acme-finished-{certname}.target` with
`acme-order-renew-{certname}.service`.

In nixos-26.05, an unset `security.acme.defaults.validMinDays` derives renewal
from lifetime: certificates lasting at least ten days renew after two thirds
of their lifetime; shorter-lived certificates renew halfway through.

### Postfix and application secrets

`services.postfix.sslCert` and `sslKey` are removed in nixos-25.11. Configure
server chains with `services.postfix.settings.main.smtpd_tls_chain_files` and
client chains with `services.postfix.settings.main.smtp_tls_chain_files`.

Nixos-26.05 replaces `services.oauth2-proxy.clientSecret` and `cookie.secret`
with `clientSecretFile` and `cookie.secretFile`. Grafana's
`services.grafana.settings.security.secret_key` has no default; deliberately
retain or rotate the old key and inject it through Grafana variable expansion
instead of storing it in Nix.

Yggdrasil's `configFile` and `persistentKeys` options are removed. Use
structured `settings` and point `settings.PrivateKeyPath` at a PKCS #8 PEM
file. A literal `PrivateKey` is rejected to prevent store disclosure.

## Service-specific migrations

### EarlyOOM, BorgBackup, AppArmor, and rsyncd

The EarlyOOM module uses upstream's hardened systemd unit in nixos-25.05. A
`killHook` needing home-directory or filesystem access may require a
`ProtectSystem` override. Each `extraArgs` element is independently
shell-escaped, not word-split.

```nix
services.earlyoom.extraArgs = [ "--prefer" "spaced pat" ];
```

`services.borgbackup.jobs.*.extraArgs` and other `extra*Args` values are Bash
arrays in nixos-25.05. Hooks must append elements rather than concatenate a
string:

```sh
extraCreateArgs+=("--exclude" "/some/path")
```

AppArmor policy `enable` and `enforce` fields are removed in nixos-25.05. Use
the `security.apparmor.policies.<name>.state` tristate.

`services.rsyncd.settings` accepts only `sections` and `globalSection` in
nixos-25.05. Move named sections below `settings.sections` and former
`settings.global` values below `settings.globalSection`.

### Forgejo and Kubernetes

`services.forgejo.dump.age` defaults to `4w` in nixos-25.11, deleting older
dumps unless retention is overridden.

`services.kubernetes.addons.dns.coredns` is renamed to `corednsImage` in
nixos-26.05 and accepts an image package, not attributes. Its default is built
locally from `pkgs.coredns` with `dockerTools.buildImage`; use a
`dockerTools.pullImage` derivation to keep an upstream image.

## Hardware and platform

`hardware.graphics.package` can select the global Mesa version without causing
a mass rebuild (since nixos-25.05).

NVIDIA configuration in nixos-26.05 gains `hardware.nvidia.branch`, unless
`hardware.nvidia.package` overrides it, and `hardware.nvidia.moduleParams` for
modprobe settings. Nixpkgs exposes `production`, `new_feature`, and `beta`
branches. Proprietary modules moved to `nvidia_x11.mod`; Maxwell and older GPUs
must pin the retained 580 legacy driver:

```nix
hardware.nvidia.package =
  config.boot.kernelPackages.nvidiaPackages.legacy_580;
```

XFS created by `xfsprogs` 6.18 uses parent pointers and exchange-range by
default (nixos-26.05). Use a 6.18-or-newer kernel for such filesystems; GRUB 2
may not boot from them.

## Locale configuration

Prefer `i18n.extraLocales` for installing additional locales as of
nixos-25.05. `i18n.supportedLocales` remains functional but is considered an
implementation detail and warns when required locales are absent.
`i18n.defaultCharset` and `i18n.localeCharsets` select character sets globally
and per locale.

## Renamed and replacement modules

Migrate these nixos-25.11 paths:

- `programs.river` to `programs.river-classic`.
- `services.nixseparatedebuginfod` to
  `services.nixseparatedebuginfod2`.
- `services.dnscrypt-proxy2` to `services.dnscrypt-proxy`.
- `services.pds` to `services.bluesky-pds`.
- `virtualisation.lxd` to `virtualisation.incus`; the LXD module is removed.

`gnome-keyring` no longer supplies an SSH agent. Use
`services.gnome.gcr-ssh-agent.enable` from `gcr_4`; for migration compatibility
it defaults to the value of `services.gnome.gnome-keyring.enable`.
