# SensorInsure / CorpSure Insurance Integration — Technical & Business Review

- Date: 2026-07-03
- Scope: The landlord building & landlord insurance product ("SensorInsure",
  brokered by CorpSure, underwritten by SGUA) — code trace across
  `sensor-alarm-backend`, `sensor-angular`, `odoo-addons`; live prod state
  (MySQL + Odoo Postgres, read-only); business model and revival levers.
- Related systems: sensor-alarm-backend, sensor-angular, odoo-addons
  (`sg_property_agency`, `sg_rest_api`, `sg_website`), LaunchDarkly,
  CorpSure API, NEXU API (stubbed), SendGrid, `sensorinsure.com` WordPress.

## Objective

Understand what the SensorInsure integration actually is, whether it is live,
how the business model works, why it under-performed, and what would have to
change to revive it.

## Terminology — two unrelated "insurance" concepts in this codebase

1. **The product (this doc):** landlord building & landlord insurance —
   everything touching `corpsure*`, `nexu*`, `tbl_insurance_*`,
   `insurance_first` / `providerPays`.
2. **Incidental:** tradesperson public-liability certificate upload
   (`businessSettings.insurance_*` in sso-provider + mobile apps). Not related.

## Business model

**SensorInsure is a customer-acquisition engine disguised as an insurance
product.** The landlord's existing annual insurance premium is redirected to a
CorpSure-brokered / SGUA-underwritten policy; slices of that premium stream
fund every party:

1. **Landlord → SGUA (via CorpSure):** the annual premium they were paying an
   incumbent insurer anyway — discounted, because the property now carries
   monitored IoT smoke alarms + water-leak detectors (lower expected claims).
2. **SGUA → CorpSure:** broker commission on the premium.
3. **CorpSure → Sensor:** **$14.99/month per insured property** (verified in
   Odoo: products 45 "Insurance Rebate" / 46 "Monthly Insurance Opted" at
   −$14.99; weekly consolidated invoice with `quantity = properties in the
   CorpSure report`).
4. **Sensor → landlord:** the −$14.99 rebate zeroes their subscription, plus
   ~$549 of hardware "free", plus $0 annual compliance testing.

Marketing promises (verbatim from SendGrid templates `S000-I`/`S154`/`S152`):
"lowers your annual Building and Landlord premium… devices valued at $549 at
no cost to you"; "If you move forward and maintain your CorpSure policy,
CorpSure will pay your Sensor subscription, so you will pay $0 for the Sensor
service"; policy sweeteners: nil excess for tenant damage & loss of rent,
flood, $5k pet damage, drug-contamination cover.

Two funnel variants (distinct `offerType`, providerPays wins if both flags set):

- **`insurance_first`** — insurance is the *hook* to acquire a new property;
  the Sensor service rides in free.
- **`providerPays`** — property already has Sensor; insurance is upsold to make
  the existing subscription free.

Distribution is via property-manager agencies (invites carry the agency's
branding; agencies can complete the form on the owner's behalf). Strata
properties are regex-excluded (`checkPropertyStrataAndUpdateProperty`) because
building cover sits with the body corporate.

## Technical flow (code-verified 2026-07-02)

Happy path: agency flagged (`tbl_admins.insurance_first`/`providerPays`, set
via `PUT /users/update-insurance-first` / `update-provider-pays`) → invite
mints `tbl_insurance_invitations` token + SendGrid `S000-I`
(`properties.entity.ts` `landlordInvitation`/`reLandlordInvitation`) →
landlord opens Angular `insurance-quote[-first-provider-pays]/:token` →
`POST /users/properties/insurance-submission` (`submitPropertyInsurance`,
entity ~:29174) writes `tbl_insurance_submission`; with LaunchDarkly flag
`insuranceViaAPI` on, backend POSTs to CorpSure and returns a `callbackURL` →
browser redirects to CorpSure's hosted form → returns with `sensor_uuid` →
`POST /complete-corpsure-insurance` (`updateInsuranceByCorpsureApi` ~:29363)
pulls `formStatus` → external infra cron hits
`GET /users/properties/sync-corpsure-insurance-data` (IP-gated,
`syncCorpsureInsuranceData` ~:13759, LaunchDarkly `CorpsureAutoAcceptance`)
which auto-accepts `issued` submissions as the synthetic
`SENSOR_AUTO_ACCEPTANCE_USERID` (stamps `offerType`, status→ACCEPTED) →
`updatePropertyStatusOnOdoo` pushes `insuranceStatus=issued` → Odoo
`sg_rest_api` `v1_update_property_status` sets
`sale_order.insurance_offer/insurance_expires` → weekly Odoo cron
`sync_weekly_invoice_to_corpsure` (`sg_property_agency/account_move.py:218`,
`crm_cron.xml`) invoices CorpSure.

Feature flags live in **LaunchDarkly** (middleware `AuthenticateFeatureFlag`,
`auth.ts:1409`), not the DB. Active keys: `insuranceViaAPI`,
`CorpsureAutoAcceptance`.

Emails: `S000-I` invite, `S005-I` SMS reminder, `S154` offer to existing
customers, `S152`/`S152-P` last-chance reminders, `S125` PDF fallback
submission to `INSURANCE_EMAIL`.

### Dead / stubbed code found

- **NEXU enrichment is stubbed** — `getPropertyDetailsFromNexu`
  (`properties.entity.ts` ~:28664) is defined but never called; `nexuData` is
  always written as `[]`. The "pre-filled quote form" promise is under-delivered.
- **`MANUAL_CORPSURE_API` flag** defined (`constants/app.ts`) but never
  referenced.
- **Per-property Odoo invoice** `process_corpsure_handling`
  (`sg_property_agency/crm_lead.py:858`) only reachable via a commented-out
  call (line 833) — all billing is the weekly batch, with no fallback.
- **Silent failure surfaces:** `authenticateCorpSureAndGetToken`,
  `getCorpsureInsuranceData` resolve errors instead of rejecting;
  `syncCorpsureInsuranceData` has an empty outer catch and aborts the whole
  loop on the first non-200 — one bad UUID can silently skip acceptance for an
  entire run. `authenticateCorpSureAndGetToken` never resolves on a 200 with no
  token (hang).
- **Env drift:** consumed vars `CORPSURE_LOGIN_API`, `CORPSURE_LOGIN_USERNAME`,
  `CORPSURE_LOGIN_PASSWORD`, `CORPSURE_ISURANCE_DATA_URL` [sic],
  `CORPSURE_TERMS_CONDITIONS_URL`, `INSURANCE_EMAIL`, `NEXU_API_URL`,
  `NEXU_API_TOKEN` are absent from `.env.example`.
- **Unverified:** possible invite-URL inconsistency — `reLandlordInvitation`
  routes insurance-first owners to `/insurance-quote/...` but
  `landlordInvitation` appears to keep `/owner?u=token` for new invites.

## Live state (read-only, measured 2026-07-02)

MySQL (`sensor-prod` via API node):

| Signal | Value |
|---|---|
| Agencies flagged | 53 `insurance_first`, 36 `providerPays` (of 56,477 admin rows) |
| Invitations | 3,787 total (2024-08-21 → 2026-06-23) |
| Submissions | 304 total; by `formStatus`: null 180, pending 71, approved 46, **issued 2**, declined 5 |
| Issued policies | 2, both created 2025-03 (property 10181 accepted cleanly; property 9015 stuck `status=11`, `offerType=NULL` — the silent-failure surface in real data) |
| Last submission | 2026-05-03 (abandoned draft) |
| Sync cron | LIVE — 127 rows re-stamped 2026-07-02 14:30 |

Invitations/month: 2025-12: 163 → 2026-01: 194 → 02: 129 → 03: 88 → **04: 4**
→ 05: 12 → 06: 15. Activity collapsed around April 2026.

Odoo (`sensorglobal_live`):

- Cron "Accounting: Weekly corpsure data - SaaS sync" **active** (last
  2026-06-30, next 2026-07-07) — but running to no effect.
- `sale_order.insurance_offer` on 167 orders (latest expiry 2026-04-01).
- **`res_company.insurance_contact` is NULL on every company** — the weekly
  invoice has no payee. Company 1 has `corpsure_data_endpoint` +
  `insurance_product` set; contact partner missing.
- Insurance-product invoices ever posted: **9** (2024-09-11 → 2025-07-11).

**Verdict: fully wired, schedulers still firing, commercially dormant.**
Funnel: 3,787 invites → 304 starts (8%) → 2 issued (~0.7% of starts) → 9
invoices. No policy issued since 2025-03, no invoice since 2025-07.

## Why it stalled (diagnosis)

Two distinct leaks:

1. **Invite → start (92% loss) = timing/relevance.** Insurance switching only
   happens in the ~4-week window around the landlord's renewal date; invites
   fire at an arbitrary moment (whenever the agency onboards the property).
2. **Start → issue (99.3% loss) = friction/trust.** Redirect out to CorpSure's
   hosted form; questions landlords can't answer offhand (rebuild cost,
   construction type — the NEXU pre-fill that would answer them was never
   wired); single-broker switch = trust hurdle.

The model's binding constraint was never the technology.

## Improvement levers (if revival is considered)

**Development direction (decided 2026-07-04):** follow-up work will be driven
from the SaferHomes side (safer-ops), not Sensor — using the Sensor product
as-is where possible and changing it only where tailoring requires. That
favours the established JV-safe pattern (see the reports tab / operational
column work): new additive Sensor endpoints (Class A) + safer-ops UI/schedulers,
over invasive edits to original Sensor flows (Class C, `properties.entity.ts`
is shared core).

Ordered by leverage; (a) and (b) are cheap and code-adjacent:

- **(a) Sell at renewal, not at invite.** Capture each property's policy
  renewal date (the sync already writes `Properties.currentPolicyExpiryDate`;
  the form already asks `current_insurer`) and re-key the existing S152
  reminder machinery to fire a personalised quote ~45 days before renewal.
  Converts blast-on-invite into drip-on-expiry.
- **(b) Kill the funnel break.** Wire the stubbed NEXU enrichment so
  rebuild-cost/construction/build-year are pre-filled; show a side-by-side
  "your current premium vs this quote" comparison instead of a bare quote link.
- **(c) Pay the channel.** Agencies earn ~nothing per conversion today. A
  per-issued-policy referral fee, or folding the offer into the annual
  smoke-alarm compliance touchpoint (the one natural landlord/safety/spend
  moment each year).
- **(d) Make telemetry earn recurring value.** Year-2 "monitored,
  zero-incident" renewal discounts (retention hook incumbents can't match);
  claims fast-track using the existing per-property audit trail; pitch Sensor
  to insurers as loss-ratio infrastructure, not lead-gen — that justifies
  richer economics than $14.99/mo.
- **(e) Widen the product.** Strata landlord-contents-only variant (building
  cover is body-corporate, but landlord cover is switchable); multi-insurer
  comparison panel (turns the trust objection into the value prop); tenant
  contents insurance on the same rails.
- **(f) Prerequisite hygiene.** Fix silent catch/abort in the sync; set
  `insurance_contact`; stop destroying consumed invites (funnel analytics are
  currently unrecoverable); add funnel-stage logging — otherwise a relaunch
  can't distinguish a timing problem from a bug.

## Risks Or Constraints

- Commercial terms (CorpSure↔SGUA commission, contract status) are not in the
  repo; premium/commission figures above are market-typical estimates. Whether
  the CorpSure relationship is even still alive is a business question.
- Any change to the invite/acceptance code is Class-B/C on original Sensor
  surfaces (`properties.entity.ts` is shared core) — regression-guard rules
  apply.
- `sensorinsure.com`/`.com.au` domains expire ~Sep 2026 (see
  `2026-04-10-migration-plan.md` §6.9).

## Open Questions

- Is the CorpSure agreement still active? Who owns the relationship post-JV?
- What killed volume in April 2026 — deliberate wind-down or attrition?
- Verify the `landlordInvitation` new-invite URL inconsistency (dead-end for
  brand-new insurance-first landlords?).
- Why is `insurance_contact` NULL — was it ever set, or did weekly billing
  always no-op after the 2025-07 invoice?

## Sources Used

- Repository files: `sensor-alarm-backend` (`properties.entity.ts`,
  `insuranceSubmission.ts`, `insuranceInvitations.ts`, routes/controllers,
  SendGrid template exports), `sensor-angular` `landlord-invite/*`,
  `odoo-addons` `sg_property_agency`/`sg_rest_api`/`sg_website`.
- Prod reads: `scripts/diag/sql-read.py` on API node (tbl_admins,
  tbl_insurance_submission, tbl_insurance_invitations, tbl_properties);
  psql via SSM on the Odoo box (ir_cron, product_template, account_move,
  sale_order, res_company). Read-only throughout.
- Prior docs: `2026-04-10-migration-plan.md` §6.9, `archive/gaps-working.md`.
