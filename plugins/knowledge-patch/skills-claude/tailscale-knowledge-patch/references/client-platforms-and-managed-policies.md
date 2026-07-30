# Client Platforms and Managed Policies

## Apply cross-platform managed connection policy

The `Hostname` system policy overrides the device hostname reported by the
operating system (1.80.0).

Windows, macOS, and iOS support `AlwaysOn.Enabled` and
`AlwaysOn.OverrideWithReason`; macOS and iOS deprecate `ForceEnabled`
(1.84.0). On Windows, Always On connects at sign-in and remains active without
the GUI, including on a headless machine. Installing the Windows client also
starts its GUI for every user who is currently signed in.

`ReconnectAfter` limits how long a user may keep Tailscale disconnected. It is
available on Windows and Android (1.84.0) and on macOS (1.86.0). Pair a managed
disconnect workflow with `tailscale down --reason` when the reason must be
recorded or enforced.

On Windows, `EnableDNSRegistration` controls whether Tailscale addresses are
registered in Active Directory DNS (1.84.0).

## Manage exit-node and subnet-routing behavior

- Android can be configured as a subnet router from its Settings menu
  (1.80.0).
- iOS provides a setting to toggle subnet routing. Version 1.84.1 fixes the
  unintended default-off state for both iOS and Android (1.84.0).
- Linux, Windows, and macOS accept `auto:any` for automatic recommended
  exit-node tracking. Windows, macOS, iOS, and tvOS expose the same behavior as
  the Recommended picker choice (1.86.0).
- On managed Windows and macOS, combine `ExitNode.AllowOverride` with
  `ExitNodeID=auto:any` to enforce use of an exit node while letting the user
  choose a different one (1.88.1).
- macOS supports the `advertiseExitNode` system policy (1.88.1).
- An iOS device can host an exit node (1.98.1).

## Configure macOS clients

The Standalone client exposes programmatic system-extension management
commands (1.80.0):

```console
tailscale configure sysext activate
tailscale configure sysext deactivate
tailscale configure sysext status
```

The client also supports these managed policies:

- `EncryptState` stores state in Keychain. The App Store build always uses
  Keychain, and Standalone v1.86.4 applies policy changes without a system
  extension restart (1.86.0).
- `OnboardingFlow` suppresses installation onboarding and replaces the
  deprecated `TailscaleOnboardingSeen` policy (1.86.0).
- `UseSystemProxy` controls whether Tailscale follows proxy configuration from
  System Settings (1.88.1).
- `AuthBrowser.macos` chooses the preferred browser for automatically opened
  authentication URLs (1.94.1).
- `HideDockIcon` controls whether the Dock icon remains after every Tailscale
  window is closed (1.94.1).
- `AppIntroShown` suppresses the Welcome to the Tailscale app modal after the
  first device login (1.98.1).

macOS 12 is the minimum supported release (1.88.1). The About screen's Release
Channel menu can opt into release-candidate builds and keep them automatically
updated (1.94.1).

Windowed UI mode is generally available. Double-click an account in the
Accounts section to switch to it (1.96.2). The open-source macOS variant
reports `node:osVersion` for posture checks, and the Standalone variant
excludes its Tailscale data directories from Time Machine backups (1.96.2).

Taildrive sharing on macOS moved from `tailscale drive` to the client GUI
(1.90.1).

## Configure Linux and other packaged clients

Linux desktop users can enable the system tray to access fast user switching,
exit-node selection, and other controls (1.88.1). Enable freedesktop autostart
with (1.96.2):

```console
tailscale configure systray --enable-startup=freedesktop
```

Linux supports TPM-backed encrypted state with `tailscaled --encrypt-state`
(1.86.0). On OpenWrt 25.12.0 and newer, client updates work when `apk` is the
package manager (1.96.2).

QNAP builds returned first as manual downloads from the Tailscale packages
site, with availability through QNAP App Center following later (1.88.1).

## Use mobile and TV authentication

iOS and tvOS clients can use auth keys against custom coordination servers,
and Apple TV can authenticate to a tailnet with an auth key. Custom
coordination servers using HTTP are accepted when their URL specifies an
explicit custom port (1.80.0).
