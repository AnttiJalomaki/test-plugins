# Networking, Security, and Native Code

## Local-network access

### Test Android 16 restrictions early

Android 16 local-network protection is opt-in. Enable the compatibility change and reboot to test it:

```shell
adb shell am compat enable RESTRICT_LOCAL_NETWORK com.example.app
```

The test gates in-process LAN TCP, UDP, multicast, broadcast, and native sockets while leaving internet traffic available. Out-of-process framework APIs such as `NsdManager` are not restricted during this test phase. Declaring and granting `NEARBY_WIFI_DEVICES` restores the gated access. Handle denial and revocation in preparation for the dedicated permission requirement.

### Request Android 17 access

API 37-targeted apps must declare and request `ACCESS_LOCAL_NETWORK`, which belongs to the `NEARBY_DEVICES` group, for LAN discovery and connections:

```xml
<uses-permission android:name="android.permission.ACCESS_LOCAL_NETWORK" />
```

An existing nearby-device grant avoids an additional prompt. When feasible, a system-mediated device picker provides a permissionless alternative. (`api-37`)

### Cross-profile loopback

Android 17 blocks loopback traffic between profiles for all apps, regardless of target SDK. Loopback within one profile is unchanged; replace cross-profile loopback protocols with a supported cross-profile mechanism.

## TLS and cleartext policy

### Encrypted Client Hello

TLS connections from API 37-targeted apps use ECH when both the networking library and server support it; otherwise they send ECH GREASE. Network Security Configuration accepts `<domainEncryption>` under `<base-config>` or `<domain-config>` to control ECH globally or per domain.

### Certificate transparency

Certificate transparency is enabled automatically for API 37 targets. API 36 required an explicit opt-in, so remove assumptions that CT remains disabled merely because no opt-in is present.

### Cleartext migration

`android:usesCleartextTraffic` is on a deprecation path and is planned to stop authorizing HTTP in a future target SDK. Move domain exceptions to Network Security Configuration. If `minSdk` is below 24, temporarily keep both mechanisms; with `minSdk` 24 or later, the network configuration alone is sufficient.

## Intent and URI launch safety

### Nested-intent redirection

Android 16 protects every app from unsafe nested-intent launches by default. If a legitimate flow breaks and the code compiles against API 36, call `removeLaunchSecurityProtection()` on the nested `Intent` only immediately before the required launch. The opt-out restores the redirection risk, so keep its scope narrow.

### Strict incoming intent matching

API 36 apps can require cross-app explicit intents to match the target component's intent filter and prevent actionless intents from matching:

```xml
<application android:intentMatchingFlags="enforceIntentFilter" />
```

A component can override the application setting with `none`; use `allowNullAction` when only absent actions must remain valid.

### Explicit URI grants

Android 18 is planned to stop implicitly granting URI access for `ACTION_SEND`, `ACTION_SEND_MULTIPLE`, and `ACTION_IMAGE_CAPTURE`. Android 17 adds `StrictMode.VmPolicy.Builder.detectImplicitUriPermissionGrant()` to find affected flows now. Add `FLAG_GRANT_READ_URI_PERMISSION` to send intents and both read and write grant flags to image-capture intents.

## Native binaries and dynamic code

### 16 KB page alignment

Android 16 can run some 4 KB-aligned apps in compatibility mode on devices using 16 KB pages and shows the user a dialog when doing so. Compiling with API 36 and enabling the `android:pageSizeCompat` manifest property suppresses the dialog, but does not replace rebuilding native code with 16 KB alignment. (`api-36`)

### Read-only native dynamic code

For API 37-targeted apps, dynamic-code-loading protection includes native libraries. A file passed to `System.load()` must already be read-only; otherwise loading fails with `UnsatisfiedLinkError`.

## Keys and signatures

### Keystore key caps

Android 17 caps API 37-targeted, non-system apps at 50,000 Android Keystore keys and other apps at 200,000. Creation beyond the limit throws `KeyStoreException`. For API 37 targets, `getNumericErrorCode()` returns `ERROR_TOO_MANY_KEYS`; older targets receive `ERROR_INCORRECT_USAGE`.

### APK Signature Scheme v3.2

Android 17 introduces APK Signature Scheme v3.2, combining an RSA or elliptic-curve signature with an ML-DSA signature for hybrid post-quantum verification.

### HPKE providers

Android 17 exposes a public service-provider interface for hybrid public key encryption implementations.
