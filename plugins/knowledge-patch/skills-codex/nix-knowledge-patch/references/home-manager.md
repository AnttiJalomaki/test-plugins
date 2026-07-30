# Home Manager

## Activation and profile management

`systemd.user.startServices` defaults to `true` as of nixos-25.05, so
activation restarts user services as needed. The `"legacy"` mode is removed
and causes an evaluation error; `"suggest"` remains temporarily available.

Use `home-manager switch --rollback` to activate the generation immediately
before the current one without creating an extra profile generation (since
nixos-25.11). Use `--specialisation NAME` to activate a named specialisation
with the same profile-preserving behavior:

```sh
home-manager switch --rollback
home-manager switch --specialisation work
```

Updating the Home Manager Nix profile from a generated activation script is
deprecated in nixos-25.11. A tool that directly calls the script must update
the profile itself. NixOS and nix-darwin module use also stops creating
per-user shadow profiles; `home-manager.enableLegacyProfileManagement = true`
temporarily restores the legacy behavior.

`home-manager.startAsUserService` in nixos-26.05 delays user activation until
login rather than system boot, supporting homes mounted later by facilities
such as `pam_mount`.

## Module loading and packaged services

`home-manager.minimal = true` imports only the basic modules required by Home
Manager (since nixos-25.11). Explicitly import every additional module:

```nix
home-manager.minimal = true;
imports = [ "${modulesPath}/programs/fzf.nix" ];
```

The `home.services` namespace introduced in nixos-26.05 lifts modular services
exported by Nixpkgs packages, such as `pkgs.<name>.passthru.services.default`,
into user systemd units without reimplementing the packaged service module.

## State-gated defaults

### State 25.05

With `home.stateVersion = "25.05"` or later, `programs.git.signing.format` no
longer defaults to `"openpgp"` (nixos-25.05). A GPG signing setup must select:

```nix
programs.git.signing.format = "openpgp";
```

### State 25.11

With `home.stateVersion = "25.11"` or later (nixos-25.11):

- Password Store again defaults to `$HOME/.password-store`, not
  `$XDG_DATA_HOME/password-store`.
- On macOS, packages' applications are copied by default to
  `~/Applications/Home Manager Apps` through
  `targets.darwin.copyApps.enable`, replacing the earlier symlink behavior.

### State 26.05 paths and XDG

With `home.stateVersion = "26.05"` (nixos-26.05):

- Zsh and Docker configuration move under XDG paths when XDG is enabled.
- Linux Firefox moves to `$XDG_CONFIG_HOME/mozilla/firefox`.
- `xdg.userDirs.setSessionVariables` defaults to `false`.
- Keys in `xdg.userDirs.extraConfig` omit the `XDG_` prefix and `_DIR`
  suffix.

### State 26.05 configuration formats

The same state version interprets Neovim plugin `config` fragments as Lua and
changes Hyprland output from Hyprlang to Lua. Pin the old Hyprland syntax when
needed:

```nix
wayland.windowManager.hyprland.configType = "hyprlang";
```

### State 26.05 automation

Automatic Home Manager upgrades no longer run `nix flake update`. Restore it
explicitly if desired:

```nix
services.home-manager.autoUpgrade.preSwitchCommands = [
  "nix flake update"
];
```

Mergiraf Git and Jujutsu integration defaults off, Yazi's shell wrapper
becomes `y`, and GTK 4 no longer inherits `gtk.theme`.

## Program and service migrations

### Syncthing

The Boolean `services.syncthing.tray` form is removed in nixos-25.11. Use:

```nix
services.syncthing.tray.enable = true;
```

In nixos-26.05, Syncthing credentials move from
`services.syncthing.passwordFile` to `services.syncthing.guiCredentials`.

### SSH

Use RFC 42-style `programs.ssh.settings` in nixos-26.05.
`programs.ssh.matchBlocks` is deprecated and automatically migrated.

The new `sshAuthSock` module provides shell integration for SSH-agent
providers. It replaces the removed
`services.ssh-agent.enable{Bash,Zsh,Fish,Nushell}Integration` options.

### Firefox, Anki, and Thunderbird

The removed top-level `programs.firefox.extensions` list moves into each
profile's `extensions.packages` or `extensions.settings` in nixos-26.05.

Anki sync settings move below
`programs.anki.profiles."User 1".sync`; `uiScale` accepts values from 1.0
through 2.0.

Thunderbird supports EWS accounts. The `outlook.office365.com` flavor defaults
unspecified IMAP and SMTP authentication to OAuth2.

### Renamed and structured modules

Migrate these nixos-26.05 changes:

- VS Code forks use dedicated modules rather than replacing
  `programs.vscode.package`.
- Select exactly one man implementation through
  `programs.man.man-db.enable` or `programs.man.mandoc.enable`.
- Rename `programs.neovim.extraLuaConfig` to `programs.neovim.initLua`.
- Rename `services.swww` to `services.awww`.
- Move assistant instruction options to their shared `context` form.
- Replace removed free-form Aerospace and aria2 configuration with structured
  `settings`.

## XDG-aware Nix and Darwin integration

`nix.assumeXdg` supports Nix installations whose base directories follow XDG
outside Home Manager (nixos-26.05). On NixOS,
`nix.settings.use-xdg-base-directories` is detected automatically.

On Darwin:

- Launchd agents wait for `/nix/store`.
- Activation replacement waits for `bootout`.
- Nix-darwin dry-run mode reaches user activation.
- `TERMINFO_DIRS` exposes terminfo supplied by packages.

These behaviors remove common ordering and environment gaps; avoid adding
competing manual workarounds.
