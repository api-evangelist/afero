---
name: Read the state of the devices on an Afero account
description: Authenticate to the Afero Cloud, resolve the caller's account, and read the live connection
  state, tags and attribute values of the devices linked to that account.
api: openapi/_original/afero-cloud-api-openapi.yml
operations:
- createAccessToken
- getCurrentUser
- listDevices
- getDevice
provider: Afero
base_url: https://api.afero.io
generated: '2026-08-02'
method: generated
---

# Read the state of the devices on an Afero account

## Authenticate first (every skill step depends on this)

1. Obtain the partner **OAuth Client ID** and **OAuth Client Secret** from the Afero Profile
   Editor under `VIEW > ACCOUNT INFO`. Never use developer credentials in production.
2. Call `createAccessToken` — `POST https://api.afero.io/oauth/token` with
   `Content-Type: application/x-www-form-urlencoded`, an
   `Authorization: Basic <base64(clientId:clientSecret)>` header, and the form body
   `username=<afero email>&password=<afero password>&grant_type=password`.
3. Read `access_token` from the response and send it as `Authorization: Bearer <access_token>`
   on every subsequent call. Tokens expire (about four hours; check `expires_in`, in seconds) —
   on any `401 unauthorized` re-run step 2 and retry once.

## Steps

1. **Resolve the account.** Call `getCurrentUser` — `GET /v1/users/me`. Read
   `accountAccess[0].account.accountId`; that is the `accountId` path parameter for every device
   call. Check `accountAccess[].privileges.canWrite` before planning any write, and
   `partnerAccess[].privileges.viewDeviceInfo` before planning any OTA work.
2. **List the devices with live state.** Call `listDevices` —
   `GET /v1/accounts/{accountId}/devices?expansions=state`. Each device returns `deviceId`,
   `friendlyName`, `deviceTypeId`, `profileId`, `partnerId`, and a `deviceState` object with
   `available`, `visible`, `connected`, `connectable`, `linked`, `rssi`, optional `location`
   and `updatedTimestamp`.
3. **Interpret connectivity before acting.** A device with `deviceState.connected: false` or
   `available: false` cannot service an attribute read or write; report it as offline rather than
   issuing a command that may never complete.
4. **Get the values.** Re-call `listDevices` with `expansions=attributes` (or `getDevice` —
   `GET /v1/accounts/{accountId}/devices/{deviceId}?expansions=attributes`) to read the
   `attributes[]` array. Each entry is `{id, data, updatedTimestamp}` where `data` is a
   HEXADECIMAL, LITTLE-ENDIAN encoding of the value — decode it against the device Profile
   before presenting it to a human.
5. **Get the labels.** Call with `expansions=tags` for `deviceTags[]`
   (`deviceTagType`, `value`, `localizationKey`) when you need the categorisation shown in the
   Afero apps.

Only one expansion is applied per request — issue one call per expansion you need.

## Conventions that apply to every step

- Base URL is `https://api.afero.io`; every resource except the token endpoint is under `/v1/`.
- Errors return `{timestamp, status, error, error_description, service_name, region}` as
  `application/json` — this is NOT RFC 9457 problem+json. See `errors/afero-problem-types.yml`.
- `id`, `versionNumber` and `firmwareImageId` are integers returned as STRINGS and may exceed
  53-bit precision — parse them with BigInt, not Number.
- Timestamps are epoch milliseconds.
- **There is no idempotency key.** Afero documents no deduplication contract, so never blind-retry
  a write; re-read state and decide. See `conventions/afero-conventions.yml`.
- OTA list operations are paged: `page` (zero-based), `size` (default 50), `sort`; the envelope
  carries `number`, `size`, `totalPages`, `numberOfElements`, `totalElements`, `content`.

## Reference

- Device API endpoints: https://afero-docs.readthedocs.io/en/latest/API-DeviceEndpoints/
- User API endpoint: https://afero-docs.readthedocs.io/en/latest/API-UserEndpoints/
