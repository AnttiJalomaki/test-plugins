# Google Play Target API Policy

Use this reference to distinguish submission eligibility from continued
discoverability. The policy facts are from the `play-target-api-policy` batch.

## New apps and updates

Beginning 31 August 2026, Google Play requires these target API levels for new
apps and app updates:

| Form factor | Minimum target API |
| --- | ---: |
| Mobile and Android Auto | 36 |
| Wear OS | 35 |
| Android Automotive OS | 35 |
| Android TV | 34 |
| Android XR | 34 |

## Existing-app discoverability

To remain discoverable to new users whose device runs a newer Android version
than the app targets, existing apps need these lower floors:

| Form factor | Minimum target API for discoverability |
| --- | ---: |
| Mobile and Android Auto | 35 |
| Wear OS | 34 |
| Android TV | 33 |
| Android Automotive OS | 32 |
| Android XR | 34 |

Below the applicable floor, an app remains available to new users only on
devices whose OS API level is no higher than the app's target. Previous
installers can still discover, reinstall, and use the app on every supported
OS version.

## Extension and exemption

An affected app can request an extension through its Play Console policy
warning or notification. An approved extension preserves full distribution
until 1 November 2026.

Permanently private apps restricted to an organisation are exempt.
Automotive-form-factor apps bundled into the same package remain discoverable
to all Google Play users.
