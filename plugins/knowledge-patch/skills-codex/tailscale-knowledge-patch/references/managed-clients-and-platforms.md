# Managed Clients and Platforms

## Device identity and authentication

- The `Hostname` system policy can override the operating system hostname
  reported by a device (since 1.80.0).
- iOS and tvOS can use auth keys with custom coordination servers, and Apple TV
  can use an auth key to join a tailnet (since 1.80.0).
- Linux, macOS, and FreeBSD 1.80.2 again accept Tailscale SSH clients that
  begin with `publickey` instead of first trying the `none` authentication
  method.
- Tailscale SSH accepts a destination IP even when MagicDNS is disabled (since
  1.88.1).
- Node-key renewal preserves existing connections while the client
  re-authenticates (since 1.90.1).
- Node-key sealing is generally available and enabled by default on Linux,
  Windows, and macOS (since 1.90.1). Existing Linux nodes migrate to sealed
  node keys automatically when upgraded.
- The `AuthKey` system policy applies only when no user is logged in (since
  1.96.2).

## Always On and disconnect policy

- Windows, macOS, and iOS provide `AlwaysOn.Enabled` and
  `AlwaysOn.OverrideWithReason` (since 1.84.0). `ForceEnabled` is deprecated
  on macOS and iOS.
- On Windows, Always On connects at sign-in and stays active without the GUI,
  including on headless systems. Installing the Windows client also starts the
  GUI for every signed-in user.
- `tailscale down` accepts `--reason` (since 1.84.0).
- `ReconnectAfter` caps how long a user may leave Tailscale disconnected on
  Windows and Android (since 1.84.0) and on macOS (since 1.86.0).
- Windows provides `EnableDNSRegistration` to control registration of
  Tailscale addresses in Active Directory DNS (since 1.84.0).

## macOS managed policy

- `OnboardingFlow` suppresses the installation onboarding flow (since
  1.86.0) and replaces the deprecated `TailscaleOnboardingSeen` policy.
- `ExitNode.AllowOverride` can be combined with `ExitNodeID=auto:any` to
  require exit-node use while permitting the user to choose another node on
  Windows and macOS (since 1.88.1).
- `UseSystemProxy` controls whether the macOS client respects proxy settings
  from System Settings, and `advertiseExitNode` is available as a macOS system
  policy (since 1.88.1).
- `AuthBrowser.macos` selects the preferred browser for automatic
  authentication URLs (since 1.94.1).
- `HideDockIcon` controls whether the Dock icon remains after all Tailscale
  windows close (since 1.94.1).
- `AppIntroShown` suppresses the Welcome to the Tailscale app modal after the
  first device login (since 1.98.1).

## Encrypted state and connection security

- The `tsStateEncrypted` posture attribute reports whether state is encrypted
  at rest (since 1.86.0).
- Linux provides TPM-backed encryption with `tailscaled --encrypt-state`.
  Windows provides the TPM-backed `EncryptState` policy.
- On macOS, `EncryptState` stores state in Keychain; the App Store client
  always uses Keychain. Version 1.86.4 applies policy changes without
  restarting the system extension.
- The 1.86.0 stable line fixes a CSRF issue that could make web-interface login
  fail and restores hostname verification when the control-plane connection
  uses a CONNECT HTTPS proxy. It also improves proxy autodetection and PAC
  handling on Windows 10 version 1607 and earlier.

## Routing controls on desktop and mobile

- Android can be configured as a subnet router from the app's Settings menu
  (since 1.80.0).
- iOS adds a subnet-routing toggle in 1.84.0. Version 1.84.1 fixes an
  unintended default-off state for subnet routing on both iOS and Android.
- Linux desktops can enable the system tray application for controls such as
  fast user switching and exit-node selection (since 1.88.1).
- An iOS device can host an exit node (since 1.98.1).

## Desktop, storage, and package behavior

- Windows 1.84.2 uses a new signing certificate with the same subject and
  issuer but a different serial number. Update deployments that allowlist the
  certificate by serial.
- Taildrive folder sharing works on Unix-like hosts without `su`, and shared
  files remain consistently accessible (since 1.88.1).
- macOS 12 is the minimum supported version as of 1.88.1.
- The macOS client no longer provides `tailscale drive` as of 1.90.1; share
  Taildrive directories through the GUI.
- The macOS About screen can select the release-candidate channel and keep
  release-candidate builds updated (since 1.94.1).
- Tailscale updates work on OpenWrt 25.12.0 and later when `apk` is the package
  manager (since 1.96.2).
- Windowed UI is generally available on macOS (since 1.96.2). Double-click an
  account in Accounts to switch to it.
- The open-source macOS build reports `node:osVersion` (since 1.96.2).
- The Standalone macOS client excludes its Tailscale data directories from
  Time Machine backups (since 1.96.2).
