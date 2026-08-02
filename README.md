# Afero

Afero, Inc. is a Los Altos, California IoT platform company founded in 2014 by Joe Britt and
Shin Matsumura. Afero ships an end-to-end connected-product stack: ASR secure radio modules with a
hardware root of trust and an embedded Hardware Security Module, the afLib MCU libraries and a
Secure Linux Device SDK, Bluetooth Low Energy and Wi-Fi onboarding, the low-code Afero Profile
Editor, Java (Android) and Swift (iOS) mobile SDKs with a Softhub, and the Afero Cloud.

- Website: https://www.afero.io/
- Developer docs: https://afero-docs.readthedocs.io/en/latest/
- Console: https://console.afero.io/
- GitHub: https://github.com/aferodeveloper
- Secondary-market listing: https://forgeglobal.com/afero_stock/

## APIs

The Afero Cloud API (`https://api.afero.io`) is the RESTful control plane for the platform —
OAuth 2.0 token issuance, the authenticated user and their account/partner privileges, device
listing with state/tags/attributes expansions, asynchronous device attribute read and write
actions, and the full over-the-air firmware pipeline (firmware types, the partner firmware pool,
binary upload and repository move, device-type associations, firmware tags, firmware push).

## A note on the OpenAPI in this repo

**Afero publishes no machine-readable API description.** `https://api.afero.io/api-docs` and
`https://api.afero.io/v1/openapi.json` both return HTTP 401 (probed 2026-08-02); no OpenAPI,
Swagger, GraphQL, AsyncAPI or MCP surface is published on any Afero host, and no A2A agent card
exists at either `/.well-known/agent-card.json` or `/.well-known/agent.json`.

The documents in `openapi/` were **derived by API Evangelist** from Afero's public developer
documentation — every path, method, parameter, payload field, response code and example is
transcribed from the Afero Developer Docs. They carry `x-apievangelist-derived-from` and
`x-apievangelist-provider-published: false` in `info`. They are not an Afero product and should
not be presented as one.
