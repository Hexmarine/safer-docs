# Property-management integrations: PropertyMe & Property Tree

**Date:** 2026-06-24
**Status:** Discovery / current-state map. Code-verified where noted; no changes made.
**Repo:** `code/sensor-alarm-backend` (integrations live entirely here, not in safer-ops)

## Objective

Map what exists today for the two external property-management (PMS) integrations
— **PropertyMe** and **Property Tree** — how they authenticate, what they sync,
in which direction, and what the known risks are. Baseline for any future work on
import reliability, incremental sync, or onboarding new agencies.

## TL;DR

Both are **one-directional imports (vendor → Sensor)** that feed the *same*
property-import pipeline used by manual CSV upload. Each integration's job is to
call the vendor API, transform the result into Sensor's internal CSV-import schema
(`PROPERTY_GUID`, `LANDLORD_NAME_1..10`, `TENANT_GUID_1..`, `AGENT_*`, `LEASE_*`),
then hand off to the shared `processUploadProperties()` importer. There is a third
source, `consolecloud`, in the same `tbl_property_files` type enum — not mapped
here.

| | PropertyMe | Property Tree |
|---|---|---|
| Direction | PME → Sensor (read-only on PME) | PT → Sensor (read-only on PT) |
| Auth | OAuth2, **per-agency** refresh token | Subscription-key + application-key → bearer, **single shared row** |
| Token store (MySQL) | `tbl_propertyme_api_tokens` (per `userId`/`agency`) | `tbl_propertytree_api_keys` (one global row) |
| Entities pulled | lots (properties), contacts (owners+tenants), members (agents) | Properties, Ownerships, Tenancies, Agents, Accounts, Agencies |
| Snapshot model | Full snapshot (`Timestamp=0`, never incremental) | Full snapshot (no delta) |
| Trigger | daily per-agency 05:00 local cron + manual | daily per-agency 05:00 local cron + manual |
| Failure handling | crash email only | Mongo **ticket** (type `PROPERTYTREE`) + email (S048) + **retry** path |
| Reconcile (deactivate missing) | **commented out** | not present |

## Sources used

- `code/sensor-alarm-backend/src/entities/properties.entity.ts` — the bulk of both
  integrations (token mgmt, fetch, transform, the shared importer).
- `code/sensor-alarm-backend/src/routes/users/v1/properties.routes.ts` — OAuth +
  sync + cron routes.
- `code/sensor-alarm-backend/src/controllers/users/properties.controller.ts`
- `code/sensor-alarm-backend/src/entities/jobLogs.entity.ts` — PT retry.
- `code/sensor-alarm-backend/src/models/mysql/propertyMeApiTokens.ts`,
  `propertyTreeApiKeys.ts`, `propertyFiles.ts`.
- `code/sensor-alarm-backend/src/constants/app.ts` — `IMPORT_SOURCE`,
  `LAST_IMPORT`, `TICKET_TYPES` enums.
- `code/pme-shim/Program.cs` — standalone PropertyMe debug/export CLI.

## Live enablement (read from `tbl_import_source`, prod, 2026-06-24)

Counts by `source` × `status` (status=1 enabled):

| source | enabled | disabled |
|---|---|---|
| `csv` | 112 | 0 |
| `propertyme` | **37** | 1 |
| `propertytree` | **13** | 1 |
| `consolecloud` | 4 | 1 |

- **PropertyMe (37 enabled)** — mostly real agencies: McGrath (Bathurst, Riverina,
  Coolangatta, Tweed Heads), Blackshaw (Manuka ×2, Weston Creek), PRD (Wagga,
  Oatley), Marq Property (×2), Blue-chip independents (Curtis & Blair, Lloyd,
  OBrien, Stone Tasmania, etc.). Two are internal (`Support - PropertyMe Agency`
  28450, `Vanessa Test Agency PropertyMe` 23738); 7 rows have a null
  `business_name` (no `tbl_business_settings` row).
- **Property Tree (13 enabled)** — Taylor & Thomas, Alice Property Management,
  Dowling Property Medowie, Blue Fox Property NSW (36058) + QLD (36245), Bartrop,
  McFall, Harcourts Signature; 6 rows null name.
- **ConsoleCloud (4 enabled) — effectively internal-only.** All four are Sensor's
  own test/sandbox agencies (`Sensor Console Prerelease Testing Agency` 13292,
  `Sensor Console Sandbox Agency` 17178, + two null). So `consolecloud` is wired
  and "live" in the enum sense but **not running for any real tenant** — treat as
  a pilot/test integration, not production. (Resolves the earlier open question.)
- **Haven (agency 37413) is not on any PMS list** — it imports via `csv`
  (the JV CSV property imports), consistent with the memory notes.

Read via `scripts/diag/sql-read.py` (read-only SELECT, SSM → prod-api node
i-0690fb2a2654ab2ad; DB creds fetched on-instance, never printed).

## Operational status — are they actually working? (2026-06-24)

Checked by reading `tbl_property_files` (every sync writes a batch row), the token
tables, the cron-box crontab, and the route middleware. Verdict per integration:

### PropertyMe — **BROKEN / silent since 2026-04-09 (~11 weeks).**

**Symptom (DB-confirmed):**
- `tbl_property_files type='propertyme'`: 13,703 successful runs all-time across 31
  agencies, then **the last row of any kind is `2026-04-09 21:00 UTC`. Zero rows
  (success *or* failure) since**, despite 37 agencies enabled.
- `tbl_propertyme_api_tokens`: all 40 rows still `authorized=1`, but the newest
  `updatedAt` is **`2026-04-09 21:00:16 UTC`** — tokens have not refreshed since the
  exact instant of the last successful sync.

**Eliminated causes (root-cause investigation 2026-06-24):**
- *Not the cron/route.* Crontab on `i-0392d1fca678f1a21` fires
  `* * * * * curl .../sync-property-me-cron`; the route (`properties.routes.ts:2314`)
  has **no auth middleware** and returns **HTTP 200 every minute** (verified in the
  structured access log), calling `syncPropertyMeCron()` fire-and-forget.
- *Not a code regression.* `git log` shows **zero commits 2026-03-25 → 2026-04-20**;
  `getTokenByRefreshToken` was last touched long before. The break is not in code.
- *Not missing config.* Runtime config is loaded by `libs/secretManager.ts`, which
  `Object.assign`s the `sensor-prod` Secrets Manager secret (159 keys) onto
  `process.env` at boot. That secret **contains all 8 `PROPERTYME_API_*` keys** (and
  all 3 `PROPERTYTREE_*`). PME is fully configured. (The on-disk `.env` is a 46-byte
  Feb-2024 stub holding only `SECRET_NAME=sensor-prod`; key names checked, values
  never read.)

**Silent-failure mechanism (code-confirmed):**
`getTokenByRefreshToken` catches any error and `return`s `undefined`; back in
`getPropertyMeToken`, `response.data.refresh_token` then throws a TypeError that the
outer `catch` swallows (`logToConsole` only) and returns `undefined`;
`syncPropertyMeCron` does `if (!accessTokenData) continue` → agency skipped with **no
row and nothing surfaced**. PME has **no failure-ticket/email** (only PT does) — which
is why this stayed invisible for 11 weeks.

**CONFIRMED ROOT CAUSE (live probe, 2026-06-24):** a controlled refresh call to
`https://login.propertyme.com/connect/token` for a test agency (23738) returned
**`HTTP 400 invalid_client`**. `invalid_client` = PropertyMe rejected the **client
credentials**, which are authenticated *before* the refresh grant is evaluated — so this
is a shared, client-level failure, not per-agency `invalid_grant`. That is exactly why
all 37 agencies stopped at the same instant. Shape-check of the `sensor-prod` secret:
`PROPERTYME_API_CLIENT_ID` len=36 and `PROPERTYME_API_CLIENT_SECRET` len=36 (both
GUID-shaped, present and well-formed), `TOKEN_URL=https://login.propertyme.com/connect/token`,
scopes intact. So the creds are **populated and correctly shaped but no longer accepted**
— PropertyMe has **revoked / expired / regenerated the OAuth client (or deactivated the
app)** around 2026-04-09. The probe failed at client-auth and consumed no token (zero side
effects). Net effect: **37 agencies' PropertyMe data frozen since 2026-04-09.**

**Fix path (not yet executed — elevated, needs approval):**
1. Obtain fresh OAuth client credentials from **PropertyMe** (check the OAuth app status /
   regenerate `client_secret`, confirm `client_id`; ask whoever owns the PropertyMe partner
   relationship — client secrets on IdentityServer can be set to expire).
2. Update `PROPERTYME_API_CLIENT_SECRET` (and `CLIENT_ID` if changed) in the `sensor-prod`
   **Secrets Manager** secret — *elevated* mutation, "go — Secrets Manager".
3. **Restart the api nodes** so the app reloads the secret (it `Object.assign`s the secret
   onto `process.env` at boot; EC2/PM2, not envFrom — needs a process restart).
4. Re-run the refresh probe. If it now returns **`invalid_grant`**, the per-agency refresh
   tokens have *also* expired after 11+ weeks idle → **re-authorize all 37 agencies** via
   the OAuth consent flow (each re-clicks "connect PropertyMe"). If it succeeds, data flow
   resumes on the next 05:00-local cron.
5. **Durable:** add a PME health signal / failure surfacing (PME has no failure-ticket/email,
   unlike PT) so a silent client-cred death can't recur unnoticed for 11 weeks.

### Property Tree — **working, but only 5 of 13 enabled agencies.**
- Token row `tbl_propertytree_api_keys` refreshed `2026-06-23 19:00` — healthy.
- 82 successful runs / 14 days. Per-agency over the last 14 days:
  - **Healthy daily (14/14):** Taylor & Thomas (1701), Blue Fox NSW (36058), Blue
    Fox QLD (36245), McFall (39311), Harcourts Signature (55586).
  - **Stalled:** Dowling Property Medowie (8344) — last success `2026-06-13`, then
    nothing.
  - **Never sync (0 runs):** 686, Alice Property Management (3543), 10515, 18469,
    30731, Bartrop Real Estate (38265), 50695. **5 of these 7 have no
    `business_name`** in `tbl_business_settings`, so the exact-name match in
    `getPropertyTreeToken` (risk #4) can never pass → permanent silent skip. The
    other two (Alice, Bartrop) have names but still don't sync — likely a PT
    company-name mismatch or they're absent from the shared application-keys array.

### ConsoleCloud — **test-only, partly failing.** 2 internal test agencies sync OK,
1 produced 16 consecutive `status=2` (zero-record) failures on 2026-06-17. No real
tenant impact.

### Why "enabled" hid all this
Both syncs are **fire-and-forget returning 200**, and PME failures don't even write
a failure row — so nothing surfaces in the UI or to operators. The only PT safety
net (failure ticket + email) doesn't exist for PME, which is exactly why PME could
die quietly for 11 weeks. **There is no end-to-end health signal for either
integration** — this investigation is the first check.

## PropertyMe

**Direction:** PME → Sensor. No write-back (broad write scopes are requested but
unused).

**Auth (OAuth2, per agency):**
1. `GET /auth-property-me` (`properties.routes.ts:~1885`) sets a CSRF `state` in
   Redis (120 s TTL) and redirects to the PME login. Scopes requested:
   `contact/property/activity/communication/transaction read+write` +
   `offline_access` (only reads are actually used).
2. `GET /auth-property-me-callback` exchanges the code for tokens.
3. Tokens stored in `tbl_propertyme_api_tokens`, keyed **per `userId`/`agency`**
   (`refreshToken` + full token JSON + `authorized`). Refreshed when the access
   token is >30 min old (`getPropertyMeToken`, `properties.entity.ts:~16873`).
4. Both OAuth routes carry a `// TODO: #propertyme this route must be whitelisted`
   note — they are not access-controlled.

**Flow:**
1. Trigger — daily cron `GET /sync-property-me-cron` (IP-gated; fires per agency
   when the agency-local clock hits 05:00) **or** manual authed
   `GET /sync-property-me` (fire-and-forget, returns after ~5 s while processing
   continues). Cron entry `syncPropertyMeCron` ~`18636`.
2. Fetch in parallel (`getPropertyMeData`, ~`17172`): `/api/v1/lots` (properties),
   `/api/v1/contacts` (owners + tenants), `/api/v1/members` (agents). Filter to
   `PrimaryType == RESIDENTIAL`, `!DeletedOn`, `!ArchivedOn`.
3. `managePropertyMeData` (~`17245`) maps PME fields → the internal CSV schema.
4. `processUploadProperties` (~`13971`) validates (GUID present, geocode address,
   lease-date format) and writes `tbl_properties`, `tbl_admins`
   (owners/agents/tenants), `tbl_lease`, `tbl_lease_tenants`, batch-tracked in
   `tbl_property_files`. New properties → review queue + optional auto-invite.

**`pme-shim/`** is a small standalone **C# .NET CLI** — reads the refresh token
from `tbl_propertyme_api_tokens`, exchanges it, dumps properties/contacts/members.
A diagnostic/export utility, not part of the runtime.

## Property Tree

**Direction:** PT → Sensor. Same shape as PME, one-directional.

**Auth (subscription-key model, MRI/Azure APIM):**
- Env: `PROPERTYTREE_BASE_URL`, `PROPERTYTREE_SUBSCRIPTION_KEY`,
  `PROPERTYTREE_APPLICATION_KEY`.
- `getPropertyTreeToken` (`properties.entity.ts:~17055`) calls
  `/apikey/v1/application_keys/{appKey}` with header
  `Ocp-Apim-Subscription-Key` → receives `{key (bearer), company_name}`.
- Token cached in `tbl_propertytree_api_keys` via `PropertyTreeApiKeys.findOne()`
  **with no `where` clause — a single global row** (5-min cache). Data calls send
  `Authorization: Bearer {key}` + `company_id: {company_name}`.
- Agency validated by **exact string match** of PT company name vs Sensor
  `tbl_business_details.business_name`.

**Flow:** Same 5-step pattern. Cron `syncPropertyTreeCron` (~`18543`), manual
`getDataFromPropertyTree`. Parallel fetch of
`/residentialproperty/v1/Properties|Ownerships|Tenancies|Agents` (agents filtered
to role `Property Manager`); filter out `deleted`/`archived`. `managePropertyData`
(~`18104`) maps to the CSV schema → shared importer.

**Extra vs PME — failure tickets + retry:** on failure it creates a Mongo `Ticket`
(type `TICKET_TYPES.PROPERTYTREE`), emails the agency (template S048), and exposes
`reTriggerPropertyTreeData` (`jobLogs.entity.ts:~3262`) to re-run from the ticket.

## Risks / constraints (code-verified)

1. **Full-snapshot, no incremental sync (both).** Every run pulls the entire
   portfolio. This is the same blast-radius shape as the Haven mass-deactivation
   incident (see `investigations/` + memory `haven-mass-deactivation-2026-06`).

2. **PME reconcile is commented out** (`properties.entity.ts:~14366–14390`). The
   block that would set `status: INACTIVE` for properties whose GUID is
   `notIn allGUIDs` (plus `inactiveJobsOnPropertyInactive`) is wrapped in
   `/* … */`. Consequence: properties deleted/archived in PME are **not**
   auto-deactivated in Sensor — data can diverge, but the import cannot mass-kill
   a portfolio. Almost certainly a deliberate post-incident safety choice; keep it
   commented unless a guarded, reviewed reconcile is designed.

3. **PT single shared token row.** `PropertyTreeApiKeys.findOne()` (no `where`)
   means one global bearer row reused across all agencies → refresh race when
   multiple agencies sync concurrently. PME's per-agency rows don't have this.

4. **PT agency match is exact-string** on `business_name`. Whitespace/case drift
   silently breaks the sync (no fuzzy match, no override).

5. **PT secondary-tenant off-by-one** (`properties.entity.ts:18229`). The
   secondary-contact loop is `for (let i = 1; i <= tenant.contacts.length; i++)`
   reading `tenant.contacts[i]`. It never visits index `0` and reads one past the
   end (last read is null-guarded, so no crash). The primary contact is selected
   separately via `.find(is_primary)`. Net effect: if the primary is **not** at
   array index 0, the non-primary contact sitting at index 0 is silently dropped.
   Low-frequency data loss, not a crash. Class C (modifies original behaviour) if
   fixed — needs regression-guard.

6. **Fire-and-forget processing (both).** The sync route returns before
   `processUploadProperties` finishes; PME surfaces failures only via crash email,
   PT via its ticket path.

7. **OAuth routes unwhitelisted (PME)** — flagged by an in-code TODO.

## Recommended next steps

- (Done next) Read `tbl_import_source` to see which agencies actually have each
  source enabled and `status=true` — scopes the real blast radius of any change.
- If incremental sync is ever pursued: PME already accepts a `Timestamp` param on
  `/api/v1/lots` (currently hard-`0`); PT would need a last-modified strategy.
- Treat the commented PME reconcile as load-bearing-by-omission: any re-enable
  must go through the property-import safety tooling (preflight/postcheck) and
  regression-guard, given the Haven precedent.
- The PT off-by-one is a small, well-scoped fix candidate, but it touches original
  Sensor behaviour → Class C, regression-guard before any change.

## Open questions

- ~~Is `consolecloud` a live source or a stub?~~ **Resolved:** wired and enabled,
  but only on 4 internal Sensor test/sandbox agencies — no real tenant uses it.
  Its sync/transform code path is unmapped here.
- Are the PME OAuth callback routes IP/whitelist-protected at the ingress/WAF
  layer even though the in-code TODO is unresolved?
- For PT, how is the single shared token row tolerated in practice — are syncs
  serialised by the 05:00-per-timezone staggering, or is the race latent?
