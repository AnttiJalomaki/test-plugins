# Home Manager

Use this reference for activation and profile behavior, state-version defaults,
module migrations, application settings, XDG locations, and Darwin integration.

## Activation, generations, and profiles

### Home Manager service activation

Since nixos-25.05, `systemd.user.startServices` defaults to `true`, so
activation restarts user services as needed. The removed `"legacy"` value is an
evaluation error; `"suggest"` remains temporarily.

### Home Manager rollback and specialisation switches

Since nixos-25.11, `home-manager switch --rollback` activates the generation
before the current one without creating another profile generation.
`home-manager switch --specialisation NAME` directly activates a named
specialisation with the same profile-preserving behavior.

### Home Manager profile ownership

Since nixos-25.11, updating the Home Manager Nix profile from a generated
activation script is deprecated. A caller that invokes the script directly
must update the profile itself. NixOS and nix-darwin module use no longer
creates per-user shadow profiles; temporarily restore that behavior with
`home-manager.enableLegacyProfileManagement = true`.

### Home Manager login-time activation

Since nixos-26.05, `home-manager.startAsUserService` defers activation until
login instead of system boot. This supports homes mounted later by facilities
such as `pam_mount`.

### Package-provided Home Manager services

Since nixos-26.05, `home.services` lifts Nixpkgs modular services such as
`pkgs.<name>.passthru.services.default` into user systemd units without
rewriting the packaged module.

## Module loading and common integration

### Minimal Home Manager module imports

Since nixos-25.11, `home-manager.minimal = true` imports only the modules needed
for Home Manager itself. Import every additional program and service module
explicitly.

```nix
home-manager.minimal = true;
imports = [ "${modulesPath}/programs/fzf.nix" ];
```

### Home Manager SSH configuration

Since nixos-26.05, `programs.ssh.settings` provides RFC 42-style SSH
configuration. `programs.ssh.matchBlocks` is deprecated and automatically
migrated. The new `sshAuthSock` module supplies shell integration for SSH-agent
providers, replacing removed
`services.ssh-agent.enable{Bash,Zsh,Fish,Nushell}Integration`.

### Home Manager XDG and Darwin integration

Since nixos-26.05, `nix.assumeXdg` handles Nix installations that use XDG base
directories outside Home Manager; NixOS
`nix.settings.use-xdg-base-directories` is detected automatically.

On Darwin, launchd agents wait for `/nix/store`, activation replacement waits
for `bootout`, nix-darwin dry-run mode reaches user activation, and
`TERMINFO_DIRS` exposes package-provided terminfo.

## State-gated defaults and file locations

### Home Manager Git signing format

With `home.stateVersion = "25.05"` or later in nixos-25.05,
`programs.git.signing.format` no longer defaults to `"openpgp"`. Select it
explicitly for GPG signing:

```nix
programs.git.signing.format = "openpgp";
```

### Home Manager 25.11 state defaults

With `home.stateVersion = "25.11"` or later in nixos-25.11, Password Store
defaults to `$HOME/.password-store`, not `$XDG_DATA_HOME/password-store`.
On macOS, packages are copied to `~/Applications/Home Manager Apps` by default
through `targets.darwin.copyApps.enable`, replacing symlinks.

### Home Manager 26.05 XDG defaults

With `home.stateVersion = "26.05"` in nixos-26.05:

- Zsh and Docker configuration move below XDG paths when XDG is enabled.
- Linux Firefox moves to `$XDG_CONFIG_HOME/mozilla/firefox`.
- `xdg.userDirs.setSessionVariables` defaults to `false`.
- Keys in `xdg.userDirs.extraConfig` omit the `XDG_` prefix and `_DIR` suffix.

### Home Manager 26.05 configuration formats

With the same state version in nixos-26.05, Neovim plugin `config` fragments
are Lua, and Hyprland generation changes from Hyprlang to Lua. Set
`wayland.windowManager.hyprland.configType = "hyprlang"` to retain the former
format.

### Home Manager 26.05 automation defaults

In nixos-26.05, automatic upgrades no longer run `nix flake update`, Mergiraf
Git and Jujutsu integration defaults off, Yazi's shell wrapper becomes `y`, and
GTK 4 no longer inherits `gtk.theme`. Restore input updates explicitly when
wanted:

```nix
services.home-manager.autoUpgrade.preSwitchCommands = [
  "nix flake update"
];
```

## Program and service migrations

### Home Manager Syncthing tray migration

In nixos-25.11, the Boolean `services.syncthing.tray` was removed. Use:

```nix
services.syncthing.tray.enable = true;
```

### Per-profile Home Manager migrations

In nixos-26.05:

- Firefox extensions move from removed top-level
  `programs.firefox.extensions` to each profile's `extensions.packages` or
  `extensions.settings`.
- Anki sync settings move below
  `programs.anki.profiles."User 1".sync`; `uiScale` accepts 1.0–2.0.
- Thunderbird adds EWS accounts, and the `outlook.office365.com` flavor
  defaults unspecified IMAP and SMTP authentication to OAuth2.

### Renamed and split Home Manager modules

In nixos-26.05:

- Syncthing credentials move from `services.syncthing.passwordFile` to
  `guiCredentials`.
- VS Code forks use dedicated modules instead of `programs.vscode.package`.
- Select exactly one of `programs.man.man-db.enable` and `.mandoc.enable`.
- Replace `programs.neovim.extraLuaConfig` with `initLua`.
- Replace `services.swww` with `services.awww`.
- Move assistant instruction options to their common `context` shape.
- Replace removed free-form Aerospace and aria2 configuration with structured
  `settings`.
