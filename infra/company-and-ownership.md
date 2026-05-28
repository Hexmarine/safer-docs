# Company & Ownership Context

Who built the SensorSyn platform, who owns and operates it now, and our role.

> **Provenance:** The company-relationship facts below were provided directly by
> the Safer Homes technical lead (stakeholder context) — not derived from code
> or pre-existing docs. Asset-custody facts are cross-referenced to investigation
> notes and `applied-changes.md`. Items still unconfirmed are called out under
> "Still open".

## The short version

- **SensorGlobal** (`Sensor Global Pty Ltd`) is the **original company** that
  built the smoke-alarm platform. The hubs/sensors and their firmware were
  **often developed through third-party vendors** contracted by SensorGlobal.
- SensorGlobal has formed a **joint venture with Safer Homes** (GitHub org
  `SaferHomesAu`). The **JV now owns and operates the product** and **continues
  development and support of effectively the entire platform**.
- This repository's work — AWS account detach, MongoDB "JV custody" cutover,
  infra inventory, integration analysis — is the **technical-side handover** from
  SensorGlobal to the JV. We are working on it from the **Safer Homes technical
  side**.
- **Post-JV people:** on the SensorGlobal side, only **CEO Andrew Cox** remains
  (the founder and the rest of the original ~15-person org per the SOC 2 System
  Description — Cameron Davis, Tom McEvoy, Sam Collins, etc. — are no longer
  involved). **Safer Homes now does all maintenance, development, and support**,
  including organising the support function. Treat any contact named in the
  pre-JV `docs/upstream/` docs as out-of-date.
- **Appinventiv (former dev/MSP vendor) is fully separated from the JV** and
  should retain **no access** to production or any system — see Entities below
  and `gaps.md` §4.

## Entities

| Entity | Role |
|---|---|
| **SensorGlobal** (`Sensor Global Pty Ltd`) | Original developer & operator of the platform. Owns the `SensorGlobal` GitHub org and, historically, the production cloud estate. |
| **Safer Homes** (`SaferHomesAu`) | JV partner. Owns the `SaferHomesAu` GitHub org (e.g. `safer-ops`). |
| **The JV** (SensorGlobal + Safer Homes) | Now owns and operates the product; continues all development & support. Referred to in docs as "the JV" / "JV-controlled". |
| **Ingram Micro** | Former **AWS reseller / billing parent** only — *not* a product developer. Production AWS account `747293622182` was detached from Ingram Micro's AWS Organization (mgmt account `872515272604`). |
| Third-party vendors | SensorGlobal contracted external vendors for hub/sensor hardware & firmware (specific vendors not yet identified). |
| **Appinventiv** (former software-dev / MSP vendor) | **Separated from the JV — no ongoing relationship.** Historically held production access + ran the `sensorglobal.msp@appinventiv.com` on-call (see `upstream/extracted/SENSORGLOB-DevOps…`). **All Appinventiv access must be revoked** (it should not reach production or any system); Safer Homes has taken over maintenance/dev/support. Tracked in `gaps.md` §4 / `gaps-detailed.md` §10. |

## Asset custody (handover status)

| Asset | Status |
|---|---|
| **AWS account `747293622182`** | Detached from Ingram Micro's AWS Org; now a standalone account under JV control — `investigations/2026-04-28-aws-account-detach-analysis.md`, `applied-changes.md`. |
| **MongoDB** | Production data cut over to a JV-controlled Atlas cluster — `runbooks/11-mongo-custody-and-aws-restore.md`, `applied-changes.md`. Original Atlas org is `Sensor Global Pty Ltd`. |
| **Domains** | 14 zones in AWS Route 53 Registrar; transfer with the AWS account — `gaps.md`. |
| **GitHub** | Two orgs in play: `SensorGlobal` (backend, SSO, Angular, etc.) and `SaferHomesAu` (`safer-ops`). |
| **Third-party SaaS** (KORE SIM, Twilio, SendGrid, Sentry, LaunchDarkly, Firebase/GCP, Bitrise) | Account ownership not yet mapped — `gaps-detailed.md` §4. |

## Hardware & firmware

- The smoke-alarm **hubs and sensors, and their firmware, were developed — often
  via third-party vendors — for SensorGlobal**. They were not built in-house in
  this codebase.
- `code/firmware` is an **empty repo**; firmware source is held by the
  hardware/firmware vendor(s), not in this repository.
- `docs/upstream/` holds **vendor-provided** technical PDFs (firmware & protocol
  spec, device types, event types, etc.), cross-referenced in
  `investigations/2026-04-12-upstream-tech-docs-review.md`. These are the
  authoritative device-side reference until firmware source/contacts are
  obtained.

## Still open

- **JV legal structure** — ownership split, which entity legally holds which
  asset, formation date — not documented.
- **Hardware/firmware vendor identity**, firmware source repo, build/OTA
  pipeline, and version tracking — `gaps-detailed.md` §6, `questions.md` Q15.
- **Third-party SaaS account ownership** — `gaps-detailed.md` §4.
