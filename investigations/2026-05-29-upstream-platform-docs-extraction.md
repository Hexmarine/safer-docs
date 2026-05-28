# Upstream platform docs — extraction & review (2026-05-29)

- Scope: the **7 top-level `docs/upstream/` docs + the Installation Process PDF** that
  the earlier [`2026-04-12-upstream-tech-docs-review.md`](2026-04-12-upstream-tech-docs-review.md)
  did **not** cover (that note handles the 12 `Technical Documents/` PDFs).
- All upstream binaries (`.docx`/`.pdf`/`.doc`) were converted to searchable text under
  [`../upstream/extracted/`](../upstream/extracted/) (pdftotext / libreoffice). The binaries
  remain authoritative; the `.txt` copies exist so the contract is grep-able and linkable.
- The firmware spec already has a curated Markdown extraction in
  [`../../code/safer-ops/docs/investigations/firmware-protocol-spec.md`](../../code/safer-ops/docs/investigations/firmware-protocol-spec.md) —
  that is the canonical version; the `.txt` here is just the raw dump.

> **Provenance caveat.** These are **original SensorGlobal documents written pre-JV**
> (authors/contacts/orgs predate the Safer Homes handover). Treat named people,
> vendors, and "mandatory/always" control claims as *as-written-then*, not
> confirmed-current. Suspected-stale items are flagged ⚠️ and collected at the end
> under **Needs confirmation** — confirm before relying on them.

---

## Most actionable finding: the install/pairing gesture (→ ties to #102)

`Technical Documents/SENSORGLOB-Installation Process` documents the on-site native
flow and the **alarm pairing-mode gesture**:

- **Double-tap the alarm's physical TEST button within ~1 s → pairing mode; the
  orange LED flashes for ~2 minutes.** Then scan/add in the app; the alarm beeps on
  success. (Matches the firmware spec's "double-press the test button" and adds the
  1-second cadence + 2-minute window + orange-LED tell.)
- Hub: scan QR → blue comms LED (far right) flashes → **solid blue after ~3 min**;
  SIM activates and the API polls `VERIFY` for **5–10 min** until the hub answers.
  Hub nominally mounted "behind the fridge."
- Test: API sends `TEST`; each alarm sounds **in sequence**; positive response → sign off.

⚠️ **This doc's header says it is superseded** by **`Smoke Alarm Manual.docx`** and
**`Sensor Hub Manual.docx`** on **OneDrive → "User Manuals"**. Those newer manuals
(and presumably a **Water Leak / A004 manual**) are the place to confirm the
**A004 water-leak pairing gesture** that bench-testing couldn't trigger (#102). We
don't have them in-repo. → **Needs confirmation #1.**

---

## Per-document notes

### `Sensor Technical Brief.docx` — concise, still-useful architecture summary
Durable facts: AWS-hosted, LB + autoscaled API, **web served statically from S3**,
Route 53 DNS; monolithic **Node.js + Angular**; native **Swift/Kotlin** apps; **MySQL
on RDS**; **MongoDB + Kafka** for event logging; **SendGrid** email; **Odoo 16
Enterprise** CRM. Hardware: hub does **433 MHz FSK RF + LTE Cat-M**, ~48 h battery;
alarms have twin 10-yr lithium cells; **KORE Super SIM**; **217E-02 smokes interconnect
without the hub**; hub-and-spoke topology. Process: SOC 2, ISO 27001 in progress,
2-week sprints, **Bitrise (mobile) + CodePipeline (API/web)**, blue-green, SonarQube/
GitGuardian, **yearly pen test by Triskele Labs**, 1-yr log retention.
- 🔑 **No OTA updates on any Sensor device** — relevant to firmware-in-transit
  concerns (#21) and to "can we patch the fleet" planning.
- ⚠️ Says **"distributed HiveMQTT backend"** — actual prod broker is **EMQX 5.8.7**
  (`2026-05-03-emqx-access-and-dashboard-credentials.md`). Marketing/stale.
- ⚠️ Says **RapidSSL wildcard** — the cert that expired 2026-04-09 was **DigiCert**
  (`2026-04-12-certificate-expiry-incident.md`).

### `Sensor Global System Description - Rev 6 - July 28 2025.docx` — SOC 2 system description
Mostly SOC 2 boilerplate (control environment, change mgmt, HR, subservice = AWS).
Useful org baseline: founded **2021 by Cameron Davis**, West Gosford NSW; **~15 staff**;
product = "Sensor Ecosystem" (Hub, Smoke Alarm, Water Leak Detector).
- ⚠️ **Org/personnel pre-JV:** CEO **Andrew Cox**, COO **Tom McEvoy**, Software Dev
  Manager **Sam Collins**. Almost certainly changed under the JV. → confirm.
- ⚠️ **Contradiction:** claims a **PaaS** that "automatically replaces failed
  containers" with HTTPS-only ingress. Our findings: **ClickOps EC2 + CodeDeploy
  blue-green**, no container PaaS (`infra/infrastructure-provisioning.md`).
- ⚠️ "No significant changes/incidents in the period" — a SOC review-window statement,
  not a current truth (cf. the April cert + SMS incidents).
- Mentions a subservice org **"TechNine"** — unexplained elsewhere; → confirm role.

### `SENSOR ARCHITECHTURE DOCUMENT_JAN2025.docx` — ~95% generic tutorial filler
Written by "Jeetendra", 23 Jan 2025, off a Dec-2024 training session. Almost entirely
copy-pasted "what is Express/MySQL/Mongo/Kafka/Redis/S3/Lambda/Security-Groups/Odoo"
definitions. Project-specific facts worth keeping (mostly already in
`2026-05-15-backend-monolith-architecture.md`):
- MVC Express app, **both** Mongoose (Mongo) and Sequelize (MySQL).
- **Redis** buffers device logs/events before Mongo, plus **flapping-event logs** and
  **trader postcode data**.
- **MongoDB** stores all system logs incl. MQTT hardware logs.
- **Socket.IO** drives dashboard/jobs/alert/notification live counts.
- **Lambda** zips PDFs (connection certificate, invoices, compliance) and emails them.
- Low net-new value beyond those bullets.

### `SENSORGLOB-Ubiquitous Language.pdf` — terminology glossary (useful)
Maps directly onto the safer-ops **installer identity model**:
- **TS** = Trade Supplier (a.k.a. Service Staff) — the individual tradesperson.
- **Contractor** = Master Trade Supplier / Trade Person / **MTS** — owner of a trade company.
- **PM** = Property Manager = Agent. **Agent Super User** = agency owner/principal.
  **Agent Admin** = a PM who can administer other agents.
- "Sensor Cloud" = backend; the SaaS portals = Agent/Owner/Trade portals ("Sensor Live").

### `SENSORGLOB-Information Security Overview.pdf` — security posture + PII inventory
PII collected: IPs, location, contractor licence/insurance numbers, addresses, phones,
names, email, company details, **ABN, contractor bank BSB + account number**, timezone.
Explicitly **not** collected: driver's licence, passport, health insurance. Hosted in
**AWS Australia**; dev access via **VPN**; pen test **Triskele Labs**; continuous audit
via **Vanta**; **GitHub Dependabot**.
- ⚠️ Claims **"2FA mandatory for accessing customer data"** and **"MFA enforced across
  all infrastructure accounts."** Our auth finding: 2FA on **7 of 56,133 portal users**
  (`2026-04-12-auth-architecture.md`). The claim may scope to admin/infra only —
  reconcile portal vs admin vs infra MFA.
- 🔑 Contractor **bank account numbers** are stored PII → relevant to data-custody/erasure
  gaps and the migration's data-protection scope.

### `SENSORGLOB-DevOps.pdf` — MSP / on-call (Appinventiv)
Production emergencies route to **sensorglobal.msp@appinventiv.com** (Freshservice ITSM,
auto-ticketing). Escalation: L1 **Pawan Kumar Binjola** (Service Delivery Lead), L2
**Kuber Singh** (Delivery Manager) — both @appinventiv, IST, 24/7. Fortnightly infra reviews.
- 🔗 Directly tied to **gaps.md #4 "Appinventiv vendor retention"**. ⚠️ Under the JV/
  account-detach this MSP relationship is likely ending → confirm whether this on-call
  path is still live, and who replaces it.

### `SENSORGLOB-Company Tools.pdf` — SaaS/tooling inventory
Teams + Outlook 365; Miro/Figma/InDesign; **Office 365, Confluence, Jira, 1Password,
OneDrive**; SendGrid, **Twilio (SIM)**, CodeTwo (email sigs), AWS, **Google Play, Apple
App Store Connect, Google Cloud (supplementary API hosting), Tricentis Testim**.
- 🔗 Maps onto **gaps.md #1** access list (Confluence/Jira, GCP, Apple/Google Play,
  1Password). Access contact listed as **Sam Collins (samc@sensorglobal.com)** — ⚠️ pre-JV.
- Note **OneDrive** is the documentation store — consistent with the User Manuals pointer above.

---

## Confirmed with the Safer Homes lead (2026-05-29)

1. **OneDrive "User Manuals" — NOT accessible.** The successor manuals
   (`Smoke Alarm Manual.docx`, `Sensor Hub Manual.docx`, any Water Leak manual) can't be
   retrieved right now, so the **A004 pairing gesture (#102)** still needs another route —
   vendor contact, the native app's behaviour, or bench observation. The Installation
   Process doc's smoke-alarm gesture (double-tap TEST within ~1 s → orange LED ~2 min) is
   the best in-hand reference; A004 may differ.
2. **Org/personnel — only CEO Andrew Cox remains** on the SensorGlobal side. Cameron Davis,
   Tom McEvoy, Sam Collins and the rest are no longer involved. **Safer Homes does all
   maintenance/development/support.** Every contact named in the pre-JV upstream docs is
   out-of-date. (Recorded in `infra/company-and-ownership.md`.)
3. **Appinventiv — fully separated from the JV; revoke all access.** No ongoing
   relationship; the `sensorglobal.msp@appinventiv.com` on-call path is **defunct, do not
   use**. Safer Homes has taken over support. → `gaps.md` §4 (decision recorded),
   `runbooks/12-aws-access-containment-follow-up.md`.
4. **"TechNine"** — unknown and **unlikely still relevant**; treated as stale SOC-template
   carryover. No further tracking unless it resurfaces.
5. **MFA — confirmed real control gap.** Universal MFA was the intent (Security Overview),
   but only 7/56,133 users have 2FA. → recorded in `gaps.md` §5.

## Stale claims to flag upstream (low priority, external docs)
- Technical Brief: "HiveMQTT" → EMQX; "RapidSSL" → DigiCert (now Amazon-managed post-incident).
- System Description: "PaaS/auto-replacing containers" → ClickOps EC2 + CodeDeploy.
- (Already tracked in the 2026-04-12 review: firmware spec "Twilio" → KORE.)
