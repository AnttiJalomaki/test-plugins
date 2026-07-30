# Privacy, Permissions, and App Data

## Health sensors

Apps targeting API 36 must replace broad sensor permissions with granular health permissions. Replace `BODY_SENSORS` with permissions such as `READ_HEART_RATE`, and replace `BODY_SENSORS_BACKGROUND` with `READ_HEALTH_DATA_IN_BACKGROUND`. The migration also covers affected Wear OS APIs and health foreground services.

Mobile apps must declare an activity that displays the privacy-policy rationale. Without it, the health permission is revoked. (`api-36`)

## Photos and selected-media access

For API 36-targeted apps on Android 16, the Photo Picker preselects app-owned photos and videos when a user grants access only to selected media. Ownership does not guarantee continued access: the user can deselect an item and revoke access immediately. Revalidate access rather than treating app ownership as authorization.

## SMS one-time passwords

Android 17 withholds WebOTP messages from all apps except the domain-verified recipient and exempt handlers for three hours, regardless of target SDK. API 37-targeted apps also lose immediate access to ordinary OTP-bearing SMS. During the delay, both `SMS_RECEIVED_ACTION` delivery and SMS provider queries are filtered. Use SMS Retriever or SMS User Consent for OTP flows. (`api-37`)

## Contacts

For API 37-targeted apps, `ContactsContract.Data` no longer exposes `ACCOUNT_NAME`, `ACCOUNT_TYPE`, or `ACCOUNT_TYPE_AND_DATA_SET`. Query `RawContacts` and join on `RAW_CONTACT_ID` when those fields are required.

Queries against `ContactsContract.Data` without `READ_CONTACTS` also enforce `StrictColumns` and `StrictGrammar`; incompatible SQL is rejected with an exception.

## Session-scoped location

Jetpack can embed a system-rendered location button that grants precise location for the current session without a permission dialog. Declare `USE_LOCATION_BUTTON` before using the control.
