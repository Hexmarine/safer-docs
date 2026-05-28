# Authentication Architecture Investigation

**Date:** 2026-04-12  
**Method:** Live MySQL introspection (SSM tunnel), Secrets Manager, prior code-component review  
**Status:** Current-state mapped; several concerns documented

---

## Overview

There are **two separate auth flows** depending on who is authenticating:

| Actor | Auth mechanism | Session store |
|---|---|---|
| Portal users (Angular web apps) | Custom OIDC provider → RS256 JWT | MongoDB (grants/clients) + MySQL (user records + JWT blocklist) |
| Mobile app users | Direct API session | MySQL `tbl_sessions` |
| Machine-to-machine (console/cloud) | OAuth refresh tokens | MySQL `tbl_console_cloud_api_tokens` |
| PropertyMe integration | OAuth per-agency refresh tokens | MySQL `tbl_propertyme_api_tokens` |

---

## 1. Portal / Web App Auth (SSO/OIDC)

### Infrastructure

```
Browser → auth.sensorglobal.com
         → sso-api-lb ALB
         → prod-sso-api EC2 (i-0d120e1ad6a527ea0, t3.small, port 4100)
         → issues RS256 JWT
         → returned to browser (stored in local/session storage)

API call → api.sensorglobal.com
         → SmokeAPI-LB ALB
         → prod-api-server EC2 (×5)
         → validates JWT signature + checks tbl_blocklist_tokens
         → resolves user from MySQL os_business_users
         → enforces tbl_role.privileges JSON
```

### SSO provider

- Repository: `sso-provider`
- Runtime: Node.js, port 4100
- OIDC backing store: **MongoDB Atlas** (`sensor-prod.vr48f.mongodb.net`) — stores OIDC clients, grants, authorization codes
- User identity source: MySQL `os_business_users` (56,133 records)
- Token format: **RS256 JWT**, 546 characters, structure `eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.eyJ…`

### Notable config concern: localhost issuer

```
OIDC_ISSUER=http://localhost:4100
```

The JWT `iss` claim will be `http://localhost:4100`, not `https://auth.sensorglobal.com`. This means:
- Standard OIDC discovery (`/.well-known/openid-configuration`) is broken for external consumers
- Any client using the issuer URL to fetch the JWKS will fail
- Works only because the API validates tokens internally with a hardcoded key, not via OIDC discovery
- **Migration risk:** must update issuer to the public URL and reissue all active sessions

### Token revocation

Logout pushes the full JWT into `tbl_blocklist_tokens`. On each API request, the token is checked against this table.

- `expiryDate` is NULL for all records — tokens are **never purged** from the blocklist
- This table will grow indefinitely; adds a DB lookup to every authenticated API call

---

## 2. User Accounts and Roles

### Portal users: `os_business_users` (56,133 records)

| Role | User Type | Active count |
|---|---|---|
| tenant | Tenant | 35,614 |
| Asset Owner | land lord | 19,693 |
| agent | Agent | 280 |
| System Administrator | Trade Person | 169 |
| Agent Admin | Agency | 102 |
| Tradesperson | Service Staff | 95 |
| agency | Agency | 64 |
| Office Staff | Trade Person | 37 |
| Admin | Super Admin | 5 |
| Sensor Admin | (none) | 4 |

Key fields: `id`, `email`, `GUID` (OIDC subject — 55,604 populated), `uu_id` (all 56,133), `Role`, `UserType`, `roleId`, `status`, `enable2fa`, `mailOTP`, `phoneOTP`, `token`

### Roles and privileges: `tbl_role` (13 roles)

| ID | Name | Code | Notes |
|---|---|---|---|
| 1 | Admin | admin | Full platform access |
| 2 | agency | agency | Agency management, properties, jobs, alarms |
| 3 | agent | agent | Properties, jobs, alarms (limited) |
| 4 | System Administrator | tradeperson | Contractor admin — jobs, costing, invoices |
| 5 | Tradesperson | servicestaff | Field worker — jobs only |
| 6 | tenant | tenant | Minimal — no property/alarm/job access |
| 7 | Asset Owner | landlord | Property/agreements/invoice view only |
| 9 | Office Staff | officestaff | Jobs + comms, no financials |
| 10 | Office Admin | officeadmin | Jobs + financials |
| 11 | Agent Admin | agentadmin | Agency admin with elevated property access |
| 12 | agency readonly | agencyreadonly | View-only across agency resources |
| 13 | Contractor readonly | contractorreadonly | View-only across contractor resources |
| 41 | Sensor Admin | sensoradmin | Platform-level (alarm mgmt, not user mgmt) |

Privileges are a per-role JSON object with resource → action → boolean structure:
```json
{ "alarms": { "view": true, "add": true, "test": true, "battery": false }, ... }
```

### 2FA

Only **7 of 56,133** users have `enable2fa=1`. OTP codes are stored as plaintext in `mailOTP` / `phoneOTP` columns of `os_business_users` — no dedicated expiry column visible in the schema.

### Password reset / email verification tokens

The `os_business_users.token` column holds a ~52-character custom token (alphanumeric + embedded Unix timestamp, e.g. `q5jr2cxrvpl1740639665085agewxm`). This is **not** the session JWT — it is a one-time token for password reset or email verification flows.

---

## 3. Mobile App Auth

### Tables

- `tbl_users`: one row per mobile device. Fields: `deviceType`, `deviceToken` (push token), `uniqueID`, `appVersion`, `notificationSetting`
- `tbl_sessions`: per-device session state. Fields: `userId`, `deviceType`, `deviceToken`, `isLogin`, `socketId`, `apiVersion`

### Session bloat concern

| isLogin | Count |
|---|---|
| 0 (logged out) | 3,561 |
| 1 (logged in) | 990,564 |

~710 sessions update per day. The remaining ~989,000 `isLogin=1` rows appear to be stale sessions that were never cleaned up. This is significant DB bloat.

---

## 4. Machine-to-Machine

### Console/Cloud API tokens (`tbl_console_cloud_api_tokens` — 5 records)

Per-agency OAuth refresh tokens for cloud console integration. Fields: `userId`, `agency`, `refreshToken`, `tokenResponseData`, `authorized`, `consoleCloudOfficeId`.

### PropertyMe integration tokens (`tbl_propertyme_api_tokens`)

Per-agency OAuth refresh tokens for PropertyMe (property management platform) integration. Fields: `userId`, `agency`, `refreshToken`, `tokenResponseData`, `authorized`.

---

## 5. Security Concerns Summary

| # | Concern | Severity | Location |
|---|---|---|---|
| 1 | `OIDC_ISSUER=http://localhost:4100` | Medium | `sensor-prod-sso` secret / SSO config |
| 2 | `SALT=SMOKE@@123` — weak hardcoded global password salt | High | `sensor-prod-sso` secret |
| 3 | OTP codes stored as plaintext in user row, no expiry | Medium | `os_business_users.mailOTP` / `phoneOTP` |
| 4 | `tbl_blocklist_tokens.expiryDate` always NULL — no purge | Low | MySQL `tbl_blocklist_tokens` |
| 5 | ~989,000 stale `isLogin=1` mobile sessions, never cleaned | Low | `tbl_sessions` |
| 6 | 2FA adoption: 7 of 56,133 users (0.01%) | Informational | `os_business_users.enable2fa` |
| 7 | `tbl_sms_allowed_list`: 2 Indian test numbers in prod (see SMS investigation) | Medium | MySQL `tbl_sms_allowed_list` |
| 8 | DB credentials typo: `DB_MYSQL_PASSWPRD` (not `PASSWORD`) | Medium | `sensor-prod` and `sensor-prod-sso` secrets |

---

## 6. Auth Flow Diagrams

### Portal login

```
1. User opens admin.sensorglobal.com (Angular SPA)
2. SPA redirects to auth.sensorglobal.com/authorize
3. SSO provider (port 4100, MongoDB backing) shows login form
4. User submits credentials
5. SSO looks up email in os_business_users (MySQL)
6. On match: creates OIDC grant in MongoDB, issues RS256 JWT
7. JWT returned to SPA (stored in browser)
8. SPA sends JWT in Authorization: Bearer header on every API call
9. API validates RS256 signature + checks tbl_blocklist_tokens
10. API loads user from os_business_users, resolves tbl_role.privileges
11. Request proceeds or 403
12. Logout: API inserts JWT into tbl_blocklist_tokens
```

### Mobile app login

```
1. App sends credentials to api.sensorglobal.com/api/v1/auth/...
2. API validates against os_business_users (or a separate mobile user store — TBC)
3. API creates/updates tbl_sessions row (isLogin=1, deviceToken)
4. API returns session token to app
5. App sends session token on subsequent requests
6. On logout: tbl_sessions.isLogin set to 0
```

---

## 7. Migration Implications

| Item | Risk | Note |
|---|---|---|
| `OIDC_ISSUER=http://localhost:4100` | High | Must update to public HTTPS URL before migrating SSO; all active sessions will be invalidated |
| MongoDB Atlas dependency | Medium | SSO provider requires Atlas; must migrate MongoDB alongside MySQL, or refactor |
| `tbl_blocklist_tokens` | Low | Append-only, no expiry — copy it all; size will be small |
| `tbl_sessions` stale bloat | Low | ~990K rows but only ~710/day active; safe to archive pre-2025 rows before migration |
| SALT change | High | Any SALT change requires all users to reset passwords |

---

## Related

- `docs/investigations/2026-04-09-platform-architecture-and-data-flow.md` — overall platform architecture
- `docs/investigations/2026-04-09-code-component-review.md` — sso-provider repo summary
- `docs/infra/prod-services-access.md` — SSM access to prod-sso-api instance
