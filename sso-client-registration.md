# SSO Client Registration Notes

Date: 2026-05-03

## Scope

This note records how a safer-ops SSO client should be created based on the checked-in `code/sso-provider` implementation. It is research only. No SSO provider code or production SSO configuration was changed.

## Current Findings

- No dynamic client registration or SSO admin route was found in `code/sso-provider`.
- OIDC clients are stored as Mongo `Client` records in `code/sso-provider/src/db/mongodb/models/Client.ts`.
- The model fields are:
  - `client_id`
  - `client_name`
  - `client_secret`
  - `redirect_uris`
  - `grant_types`
  - `scope`
  - `application_type`
- The Mongo adapter in `code/sso-provider/src/adapters/mongodb.ts` only reads the `Client` collection when the requested id contains `APP_CONFIG.CLIENT_SUFIX`.
- `APP_CONFIG.CLIENT_SUFIX` comes from the `CLIENT_SUFIX` environment variable in `code/sso-provider/src/configs/app.config.ts`.

## Implication For Safer Ops

The safer-ops client id must include the configured `CLIENT_SUFIX` value for the SSO provider to treat it as a client id and look it up in the `Client` collection. In the local Kubernetes fixture this value is `_client`, so a client id like `safer_ops_client` would satisfy that shape there. Production must be checked against the live SSO environment value before creating the record.

## Recommended Registration Path

Create the safer-ops client as a Mongo `Client` record by controlled DB seed or a manual Mongo insert approved by the SSO owner. Do not modify `sso-provider` code for this registration unless a broader client-management workflow is explicitly approved.

Suggested client attributes:

```json
{
  "client_id": "safer_ops_client",
  "client_name": "Safer Ops",
  "client_secret": "<generated-secret>",
  "redirect_uris": ["http://127.0.0.1:8080/auth/callback"],
  "grant_types": ["authorization_code", "refresh_token", "password"],
  "scope": "openid email profile api:read offline_access",
  "application_type": "web"
}
```

Use the real deployed callback URL for any non-local environment. Store the secret in the approved runtime secret store, then configure safer-ops with `OIDC_CLIENT_ID`, `OIDC_CLIENT_SECRET`, `OIDC_ISSUER_URL`, and `OIDC_REDIRECT_URI`.

## Open Checks Before Production Use

- Confirm the production `CLIENT_SUFIX` value.
- Confirm whether safer-ops should use a dedicated SSO client separate from the Sensor Angular/admin clients.
- Confirm whether password grant is acceptable for the service-login Sensor API connection path, or whether operators should seed a refresh token through a separate approved process.
- Confirm exact redirect URIs for local, staging, and production safer-ops deployments.
