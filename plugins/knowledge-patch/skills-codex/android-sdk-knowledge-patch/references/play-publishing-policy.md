# Google Play Target API Policy

## New apps and updates

From 31 August 2026, Google Play requires new apps and updates to target:

| Form factor | Minimum target API |
|---|---:|
| Mobile | 36 |
| Wear OS | 35 |
| Android Automotive OS | 35 |
| Android TV | 34 |
| Android XR | 34 |

## Existing-app discoverability

To remain discoverable to new users whose device runs a newer Android version than the app targets, existing apps must meet these lower floors:

| Form factor | Minimum target API for discoverability |
|---|---:|
| Mobile and Android Auto | 35 |
| Wear OS | 34 |
| Android TV | 33 |
| Android Automotive OS | 32 |
| Android XR | 34 |

Below the applicable floor, the app remains available to new users only on devices whose OS API level is no higher than the app's target. Previous installers can still discover, reinstall, and use it on every supported OS version.

## Extensions and exemptions

An affected app can request an extension through its Play Console policy warning or notification to retain full distribution until 1 November 2026.

Permanently private apps restricted to an organisation are exempt. Automotive-form-factor apps bundled in the same package also remain discoverable to all Google Play users. (`play-target-api-policy`)
