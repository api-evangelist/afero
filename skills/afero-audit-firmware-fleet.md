---
name: Audit the firmware inventory of an Afero device type
description: 'Read-only inventory of a partner''s Afero firmware estate: firmware types, pool images,
  per-device-type associations and tags, with no write operations.'
api: openapi/afero-ota-api-openapi.yml
operations:
- createAccessToken
- getCurrentUser
- listFirmwareTypes
- getFirmwareType
- listFirmwareTags
- listPoolFirmwareImages
- listPoolFirmwareImagesByType
- listPoolFirmwareImageAssociations
- listDeviceTypeFirmwareImages
- listDeviceTypeFirmwareImagesByType
- getDeviceTypeFirmwareImage
provider: Afero
base_url: https://api.afero.io
generated: '2026-08-02'
method: generated
---

# Audit the firmware inventory of an Afero device type

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

This skill is READ-ONLY. It issues no POST, PUT or DELETE. Use it to answer "what firmware exists,
and where is it deployed" before anyone runs the publish skill.

## Steps

1. **Resolve the partner.** `getCurrentUser` (`GET /v1/users/me`) →
   `partnerAccess[0].partner.partnerId`. Require `partnerAccess[].privileges.viewDeviceInfo`.
2. **Enumerate firmware types.** `listFirmwareTypes`
   (`GET /v1/ota/partners/{partnerId}/types`). For each, record `type`, `name`,
   `versionAttributeId` and whether it is a platform (1-100) or MCU (101-200) type. Drill into a
   single one with `getFirmwareType` (`GET /v1/ota/partners/{partnerId}/types/{type}`).
3. **Enumerate the tag vocabulary.** `listFirmwareTags`
   (`GET /v1/ota/partners/{partnerId}/tags`) returns the flat array of tag strings you can
   filter every listing on.
4. **Walk the firmware pool.** `listPoolFirmwareImages`
   (`GET /v1/ota/partners/{partnerId}/pool`) with `page`, `size` and `sort=updatedTimestamp`.
   Page until `number + 1 == totalPages`. Narrow to one firmware type with
   `listPoolFirmwareImagesByType` (`.../pool/types/{type}`), and filter with `tags`.
5. **Find where a pool image is deployed.** For each interesting `type` + `versionNumber` call
   `listPoolFirmwareImageAssociations`
   (`.../pool/types/{type}/versionNumbers/{versionNumber}/associations`) — it returns the
   `deviceTypeId`, `deviceTypeName`, owning `partnerId` and contact `email` for every association.
6. **Walk a device type's eligible images.** `listDeviceTypeFirmwareImages`
   (`GET /v1/ota/partners/{partnerId}/deviceTypes/{deviceTypeId}/firmwareImages`), narrowed
   with `listDeviceTypeFirmwareImagesByType` and resolved to a single record with
   `getDeviceTypeFirmwareImage` (`.../types/{type}/versionNumbers/{versionNumber}`).
   Records of firmware type 4 (`DEVICE_DESCRIPTION`) additionally carry `deviceDescriptionId`
   and `deviceProfileId`, which tie the image back to a published device Profile.
7. **Report the gaps.** Flag pool images with no associations (built but never deployable), device
   types whose newest associated image is older than the newest pool image of the same type, and
   any image whose `url` is empty (the binary was never moved into the repository).

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

- OTA endpoint reference: https://afero-docs.readthedocs.io/en/latest/API-OTAEndpoints-Funcs/
- Firmware pools and associations: https://afero-docs.readthedocs.io/en/latest/API-OTAEndpoints/
