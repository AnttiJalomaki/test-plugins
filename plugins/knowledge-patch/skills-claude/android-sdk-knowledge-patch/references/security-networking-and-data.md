# Security, Networking, and Data Access

Use this reference for native compatibility, permission, intent, network,
storage, cryptography, and profile-isolation work. Source batch IDs are shown
as `api-36` and `api-37` where behavior is API-specific.

## Contents

- [Native compatibility and dynamic code](#native-compatibility-and-dynamic-code)
- [Permission migrations](#permission-migrations)
- [Intent and component boundaries](#intent-and-component-boundaries)
- [TLS and cleartext traffic](#tls-and-cleartext-traffic)
- [Data-provider and storage behavior](#data-provider-and-storage-behavior)
- [Profile isolation and cryptography](#profile-isolation-and-cryptography)
- [SMS OTP confidentiality](#sms-otp-confidentiality)

## Native compatibility and dynamic code

### 16 KB memory pages (`api-36`)

Android 16 can run some 4 KB-aligned apps in compatibility mode on devices
using 16 KB pages and displays a user dialog when doing so. When compiling
with API 36, enabling the `android:pageSizeCompat` manifest property suppresses
the dialog. This is not a substitute for rebuilding native code with 16 KB
alignment.

### Read-only native dynamic code (`api-37`)

For API 37-targeted apps, dynamic-code-loading protection includes native
libraries. A file passed to `System.load()` must already be read-only;
otherwise loading fails with `UnsatisfiedLinkError`.

## Permission migrations

### Granular health sensors (`api-36`)

API 36-targeted apps replace `BODY_SENSORS` with specific health permissions,
such as `READ_HEART_RATE`, and replace `BODY_SENSORS_BACKGROUND` with
`READ_HEALTH_DATA_IN_BACKGROUND`. The migration includes affected Wear OS APIs
and health foreground services.

Mobile apps must declare an activity that explains the privacy-policy
rationale. If it is absent, the permission is revoked.

### Local network trial mode (`api-36`)

Android 16 local-network protection is opt-in for testing. Enable
`RESTRICT_LOCAL_NETWORK`, then reboot:

```shell
adb shell am compat enable RESTRICT_LOCAL_NETWORK com.example.app
```

The flag gates in-process LAN TCP, UDP, multicast, broadcast, and native
sockets while leaving internet traffic available. Out-of-process framework
APIs such as `NsdManager` are not restricted during this phase. Declaring and
granting `NEARBY_WIFI_DEVICES` restores the gated access. Test both denial and
revocation before relying on future dedicated permission behavior.

### Local network runtime permission (`api-37`)

API 37-targeted apps must declare and request `ACCESS_LOCAL_NETWORK`, which is
in the `NEARBY_DEVICES` group, for LAN discovery and connections. An existing
nearby-device grant prevents another prompt. A system-mediated device picker
is the alternative when direct permission is undesirable.

```xml
<uses-permission android:name="android.permission.ACCESS_LOCAL_NETWORK" />
```

## Intent and component boundaries

### Nested-intent launch hardening (`api-36`)

Android 16 protects every app against unsafe launches of nested intents. If a
reviewed legitimate flow breaks, code compiled against API 36 may call
`removeLaunchSecurityProtection()` on the nested `Intent` before launching it.
This restores the redirection risk and should not be applied broadly.

### Strict incoming intent matching (`api-36`)

API 36 apps can require explicit cross-app intents to match the target
component's filter and stop actionless intents from matching:

```xml
<application android:intentMatchingFlags="enforceIntentFilter" />
```

A component can override the application setting with `none`.
`allowNullAction` selectively permits an absent action.

### Background launches through IntentSender (`api-37`)

Android 17 extends background-activity launch controls to `IntentSender`.
Replace the broad legacy `MODE_BACKGROUND_ACTIVITY_START_ALLOWED` with a
granular mode such as `MODE_BACKGROUND_ACTIVITY_START_ALLOW_IF_VISIBLE`. Use
StrictMode or lint to locate old flows.

### Explicit URI grants (`api-37`)

Android 18 will stop providing implicit URI access for `ACTION_SEND`,
`ACTION_SEND_MULTIPLE`, and `ACTION_IMAGE_CAPTURE`. Android 17 provides
`StrictMode.VmPolicy.Builder.detectImplicitUriPermissionGrant()` to find such
dependencies.

Add `FLAG_GRANT_READ_URI_PERMISSION` to send intents. Add both read and write
grant flags to image-capture intents.

## TLS and cleartext traffic

### Encrypted Client Hello (`api-37`)

TLS connections from API 37-targeted apps use Encrypted Client Hello when the
network library and server support it; otherwise they send ECH GREASE. Network
Security Configuration accepts `<domainEncryption>` inside `<base-config>` or
`<domain-config>` to enable or disable ECH globally or by domain.

### Certificate transparency (`api-37`)

Certificate transparency is enabled automatically for API 37-targeted apps.
This differs from API 36 behavior, which required an explicit opt-in.

### Cleartext migration (`api-37`)

`android:usesCleartextTraffic` is on a path to stop authorizing HTTP in a
future target SDK. Put domain exceptions in Network Security Configuration.
If `minSdk` is below 24, keep both the manifest mechanism and network
configuration temporarily; with `minSdk` 24 or later, the network
configuration is sufficient.

## Data-provider and storage behavior

### App-specific MediaStore token (`api-36`)

For API 36-targeted apps, `MediaStore.getVersion()` returns a different value
for each app. Treat it as an opaque change token. Do not parse it or infer
device details from it.

### Restricted contacts queries (`api-37`)

For API 37 targets, `ContactsContract.Data` does not expose `ACCOUNT_NAME`,
`ACCOUNT_TYPE`, or `ACCOUNT_TYPE_AND_DATA_SET`. Query `RawContacts` and join
through `RAW_CONTACT_ID` instead.

Queries against `Data` without `READ_CONTACTS` also enforce `StrictColumns`
and `StrictGrammar`; incompatible SQL is rejected with an exception.

### Per-app Keystore limits (`api-37`)

Android 17 caps API 37-targeted non-system apps at 50,000 keys and other apps
at 200,000. Creation beyond the cap throws `KeyStoreException`.
`getNumericErrorCode()` reports `ERROR_TOO_MANY_KEYS` for API 37 targets and
`ERROR_INCORRECT_USAGE` for older targets.

## Profile isolation and cryptography

### Cross-profile loopback (`api-37`)

Android 17 blocks loopback traffic between profiles for all apps regardless of
target SDK. Loopback within one profile is unchanged.

### APK Signature Scheme v3.2 (`api-37`)

Android 17 introduces APK Signature Scheme v3.2. It combines RSA or
elliptic-curve signatures with ML-DSA signatures for post-quantum hybrid
verification.

### HPKE provider interface (`api-37`)

Android 17 exposes a public service-provider interface for hybrid public key
encryption implementations.

## SMS OTP confidentiality (`api-37`)

Android 17 withholds WebOTP messages from every app except the domain-verified
recipient and exempt handlers, regardless of target SDK. API 37-targeted apps
also lose immediate access to ordinary OTP-bearing SMS.

During the three-hour delay, both `SMS_RECEIVED_ACTION` delivery and SMS
provider queries are filtered. Use SMS Retriever or SMS User Consent for OTP
flows.
