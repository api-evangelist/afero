---
name: Control an Afero device by writing an attribute
description: Actuate a connected Afero device by writing a device attribute, understanding that the write
  is asynchronous, unacknowledged by the API response, and has real physical consequences.
api: openapi/_original/afero-cloud-api-openapi.yml
operations:
- createAccessToken
- getCurrentUser
- listDevices
- executeDeviceAction
- getDevice
provider: Afero
base_url: https://api.afero.io
generated: '2026-08-02'
method: generated
---

# Control an Afero device by writing an attribute

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

> **This skill actuates real hardware.** `executeDeviceAction` with
> `type: attribute_write` changes the physical state of a deployed device. Treat it as
> human-in-the-loop: confirm the target device and the intended value with a person before
> issuing the call. See `agentic-access/afero-agentic-access.yml`.

## Steps

1. **Resolve the account and confirm write privilege.** Call `getCurrentUser`
   (`GET /v1/users/me`) and require `accountAccess[].privileges.canWrite: true` for the account
   you are about to act on. If it is false, stop — the write will fail with `403`.
2. **Find the device and confirm it is reachable.** Call `listDevices`
   (`GET /v1/accounts/{accountId}/devices?expansions=state`) and select the device by
   `friendlyName` or `deviceId`. Require `deviceState.connected: true`. If the device is offline,
   Afero documents that the read/write request may never complete — do not queue it silently.
3. **Read the current value first.** Call `executeDeviceAction` —
   `POST /v1/accounts/{accountId}/devices/{deviceId}/actions` with
   `{"type": "attribute_read", "attrId": <id>}`, then re-read the attribute via `getDevice`
   with `expansions=attributes`. Never write without knowing the value you are replacing.
4. **Write the new value.** Call `executeDeviceAction` with
   `{"type": "attribute_write", "attrId": <id>, "data": "<hex>"}`. `data` MUST be hexadecimal
   encoded, LITTLE ENDIAN — a two-byte value of 1 is `"0100"`, not `"0001"` or `"1"`. The
   attribute must support writes; check the device Profile.
5. **Understand the response.** The `200` body is an ACKNOWLEDGEMENT OF THE REQUEST, not of the
   device: `{type, requestId, timestampMs, sender, source}`. It does not mean the device
   applied the value.
6. **Verify by polling state, not by retrying.** Re-read the attribute with `getDevice` +
   `expansions=attributes` and compare `data` and `updatedTimestamp`. Because there is no
   idempotency key, a blind retry may double-apply a relative or toggling attribute — poll,
   then decide.

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
- Attribute modelling and value change rules: https://afero-docs.readthedocs.io/en/latest/AttrModel/
