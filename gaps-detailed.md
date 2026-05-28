# Information & Access Gaps

Consolidated list of **systems we need access to** and **operational questions to put to the original product/operations owners** in order to complete the SensorGlobal platform walkthrough and unblock migration planning.

Each gap entry lists:
- **What we need** — concrete artefact, credential, or answer
- **Why** — what investigation or decision it unblocks
- **Likely owner** — best-guess contact (confirm with stakeholder map)

Sorted by **criticality** (🔴 Critical → 🟢 Low). Criticality reflects production safety, data-loss risk, migration-blocker status, and security exposure — *not* difficulty to answer.

> Backlog source: `questions.md` + sweep of `investigations/`, `infra/`, `runbooks/`, and `BACKUP-INVENTORY.md`. See [`gaps.md`](./gaps.md) for the compact one-line summary.

---

## Index

| # | Tier | Topic |
|---|---|---|
| 1 | 🔴 Critical | Active P0/P1 production incidents — ownership |
| 2 | 🔴 Critical | Backup & DR coverage gaps |
| 3 | 🔴 Critical | External AWS account `813668149731` — purpose & ownership |
| 4 | 🔴 Critical | Third-party SaaS account ownership map |
| 5 | 🔴 Critical | Sandbox / soapbox environment |
| 6 | 🟠 High | Firmware source / build / vendor |
| 7 | 🟠 High | EMQX MQTT broker — operational access |
| 8 | 🟠 High | Production database — read-only access for stats |
| 9 | 🟠 High | Mobile app store accounts & signing keys |
| 10 | 🟠 High | Appinventiv vendor retention decision |
| 11 | 🟠 High | Cert renewal & email-domain ownership (DigiCert→ACM, DMARC) |
| 12 | 🟠 High | Auth/security debt — intent vs accident |
| 13 | 🟠 High | Infrastructure provenance — ClickOps history |
| 14 | 🟡 Medium | Error / incident management workflow |
| 15 | 🟡 Medium | Stale / legacy data — on-purpose or forgotten? |
| 16 | 🟡 Medium | Agency impersonation / "login-as" support workflow |
| 17 | 🟡 Medium | Property audit history coverage & retention |
| 18 | 🟡 Medium | Tenant legality / compliance |
| 19 | 🟡 Medium | Lease management — operational ownership |
| 20 | 🟡 Medium | Customer-facing surfaces (support, status, T&Cs, privacy) |
| 21 | 🟢 Low | Per-contact notification routing (product decision) |

---

# 🔴 Critical

## 1. Active P0/P1 production incidents — ownership

Four live production issues remain unowned post-investigation:

| Incident | Status | Source |
|---|---|---|
| **SMS pipeline 93% drop since 2026-03-26** | Root cause is application-layer; not yet fixed | `2026-04-12-mysql-sms-root-cause.md` |
| **899 emails stuck since April 2025** in `tbl_communications_queue` | Queue processor not running? SendGrid key rotated? | `2026-04-12-current-env-maintenance-actions.md` §5 |
| **EMQX TLS cert expired on port 18756** (16 of 21 hubs use this port) | Hubs ignore expiry, but security gap | `2026-04-10-inv-results.md` INV-01 |
| **DLM Kafka snapshot tag mismatch** — no Kafka EBS snapshots since Nov 2024 | DLM targets `prod-mongodb-kafka-server`, instance is `prod-kafka` | `2026-04-17-prod-snapshot-coverage.md` |

**Why**: This is a smoke-alarm safety platform. SMS being broken means residents may not be alerted to fires. Need owner confirmation that someone is on these.

**Likely owner**: App/Platform Engineering.

---

## 2. Backup & DR coverage gaps

| Asset | Gap |
|---|---|
| **MongoDB Atlas** | Atlas-side backup policy enabled status not verified — `2026-04-17-prod-snapshot-coverage.md` |
| **Odoo filestore (75 GB)** | No portable logical export; only EBS snapshot (account-bound) — `BACKUP-INVENTORY.md` |
| **EMQX auth DB / ACL / retained messages / certs** | No `emqx_ctl data export` automation — `BACKUP-INVENTORY.md` |
| **Terraform state S3** (`sensorsyn-terraform-state-747293622182`) | No cross-region replication; lost if account is lost — `BACKUP-INVENTORY.md` |
| **Kafka data on `/tmp`** | Lost on instance reboot; risk acceptance is informal — `2026-04-12-current-env-maintenance-actions.md` §4 |
| **Redis snapshot retention = 1 day** | Acceptable for cache, risky if any durable data — `2026-04-17-prod-snapshot-coverage.md` |

**Need from owners**: Confirmation that this risk profile is **deliberate** (not an oversight), and any history of past data loss / restores.

**Likely owner**: DevOps / Platform Lead.

---

## 3. External AWS account `813668149731` — purpose

| Need | Detail |
|---|---|
| **Why does it exist?** | Three production ALBs (`prod-syd-1-az1/2/3`) live in a separate AWS account (813668149731) with expired certs |
| **Who owns it?** | Account root contact, billing, IAM admin |
| **Is it still serving live traffic?** | Or is it a legacy/unused account that should be decommissioned? |

**Why**: Cannot complete migration mapping without knowing whether this account is in scope. Expired certs in it suggest neglect.

**Likely owner**: DevOps / Platform Lead.

**Source**: `2026-04-12-certificate-expiry-incident.md`.

---

## 4. Third-party SaaS account ownership map

Across the platform there are **10+ vendor integrations** with no documented account ownership, billing contact, or escalation path. We need a mapping for each:

| Vendor | What we need to know |
|---|---|
| **KORE SIM** (cellular IoT) | Master account holder, billing, contact for SIM provisioning; 21 active hubs depend on this |
| **Twilio** (SMS) | Is Twilio still in use post-SendGrid migration? Account owner, billing |
| **SendGrid** (email) | Account owner, API key rotation policy |
| **Sentry** (error monitoring) | Owner, project list, billing |
| **LaunchDarkly** (feature flags) | Owner, flag inventory, billing |
| **Firebase / GCP** (`sensor-production` project) | GCP account owner, mobile push config, billing |
| **Bitrise** (mobile CI/CD) | Account owner, signing key custody |

**Why**: Migration to a new entity may require re-registration; vendor outages today have no clear escalation; secrets cannot be rotated without knowing who owns the account.

**Likely owner**: Product Owner + Finance/Accounts (per-vendor mapping).

**Source**: `2026-04-20-infra-inventory.md` §1.2; `2026-04-21-odoo-integrations-analysis.md`; `2026-04-10-migration-plan.md` §6–9.

---

## 5. Sandbox / Soapbox environment

| Need | Detail |
|---|---|
| **Writable non-prod environment** | Already located UAT (`2026-04-16-uat-environment-discovery.md`) but no seeded data or login credentials |
| **Test agency / agent / contractor accounts** | To exercise onboarding, hub install, alarm-fire flows end-to-end |
| **Test SIM / hub** | A spare hub with active LTE Cat-M SIM to drive real MQTT traffic |
| **Mobile app test build** | To trigger the QR-scan installation flow |

**Why**: Read-only investigation has reached its limit. Several flows (hub registration, alarm dispatch, config save) cannot be validated without an environment we can write to.

**Likely owner**: Engineering lead + Operations.

---

# 🟠 High

## 6. Firmware (Sensor Hub & device firmware)

**Known**: The hubs/sensors and their firmware were developed via **third-party vendor(s)** contracted by SensorGlobal — not built in-house in this codebase. See `infra/company-and-ownership.md`. The items below are still outstanding.

| Need | Detail |
|---|---|
| **Source repository** | Where is firmware code hosted? (Not in any repo under `code/`; the `firmware` repo is empty.) |
| **Build pipeline** | How is firmware built, signed, OTA-distributed? |
| **Versioning** | How are device firmware versions tracked vs hub records in DB? |
| **Vendor identity** | Which third-party vendor(s) built the hubs/sensors and firmware? Need an introduction. |

**Why**: Cannot reason about device-side bugs, OTA strategy, or migration impact without it. Also a security/compliance gap (Vanta).

**Likely owner**: Third-party hardware/firmware vendor (via SensorGlobal) — needs introduction.

---

## 7. EMQX MQTT broker — operational access

| Need | Detail |
|---|---|
| **Read-only `emqx_ctl` access** | To confirm live hub traffic, count connected clients, list topics |
| **Dashboard credentials** | EMQX web dashboard for active sessions and metrics |
| **Subscription audit** | Confirm `$share/sensorGroup/sg/sas/resp/+` consumers in production |
| **Retained-message and ACL inspection** | For migration risk assessment |

**Why**: Required to answer *"is there traffic from hubs?"* and to validate cutover plans. Currently we only know broker exists in `ap-southeast-2`; no live confirmation of state.

**Likely owner**: Platform/DevOps lead.

---

## 8. Production database — read-only query access

| Need | Detail |
|---|---|
| **Read-only MySQL replica / RDS access** | For alarm counts, fire-rate, hub-traffic stats |
| **Read-only MongoDB access** | For event/log timeseries (alarm events, hub heartbeats) |
| **Snapshot or anonymised dump** | For local analysis without touching prod |

**Why**: All operational stats questions depend on direct telemetry queries; web portal does not surface them.

**Likely owner**: DBA / Platform lead.

---

## 9. Mobile app store accounts & signing keys

| Need | Detail |
|---|---|
| **Apple Developer account** | Owner, billing, team membership, distribution certificates |
| **Google Play console** | Owner, billing, upload key custody |
| **App signing keys** | Where are iOS provisioning profiles & Android upload key stored? Backup? Rotation procedure? |
| **Push notification keys** | APNs key, FCM server keys — owner & rotation |

**Why**: Loss of signing keys = cannot ship app updates. No location documented in any AWS secret or runbook.

**Likely owner**: Mobile Lead / DevOps.

---

## 10. Appinventiv vendor retention decision

| Need | Detail |
|---|---|
| **Are Appinventiv still retained?** | 6 IAM users (`deepak`, `gurpreet`, `jeetendra`, `lokendra`, `rudresh`, `Appinventiv_S3`) have production access |
| **If yes** — what's the engagement model going forward; should access be rotated/restricted? |
| **If no** — when can we revoke? Any code/IP transfer outstanding? |
| **`lokendra` SG exception** — security group rule grants ALL-traffic to `116.73.36.32/32` (a personal IP); can it be removed? |

**Why**: Production access from external developers must be either formalised or revoked before migration. Currently in limbo.

**Likely owner**: Product Owner / Operations.

**Source**: `2026-04-10-migration-plan.md` §9; `2026-04-10-inv-results.md` INV-01.

---

## 11. Cert renewal & email-domain ownership

| Need | Detail |
|---|---|
| **DigiCert → ACM transition** | Following the cert expiry incident, who now monitors ACM auto-renewal? Any escalation if renewal fails? — `2026-04-12-certificate-expiry-incident.md` |
| **DMARC `p=none`** | Domain currently allows spoofing. Is this intentional, or never tightened? — `2026-04-10-migration-plan.md` INV-06 |
| **SPF / DKIM / mail-from records** | Owner of DNS records for outbound mail authenticity |

**Likely owner**: DevOps / Security.

---

## 12. Auth/security debt — intent vs accident

Several auth-system oddities — owners can tell us whether they're intentional or just legacy that nobody owns:

| Item | Source |
|---|---|
| `OIDC_ISSUER=http://localhost:4100` (not public URL) | `2026-04-12-auth-architecture.md` §1 |
| Hardcoded SSO password salt `SALT=SMOKE@@123` | `2026-04-12-auth-architecture.md` §5 |
| Plaintext OTP codes in `mailOTP` / `phoneOTP`; no expiry | `2026-04-12-auth-architecture.md` §5 |
| JWT blocklist has no TTL; grows forever | `2026-04-12-auth-architecture.md` §1 |
| 989,000 stale `isLogin=1` rows in `tbl_sessions` | `2026-04-12-auth-architecture.md` §3 |
| 2FA enabled for only 7 of 56,133 portal users | `2026-04-12-auth-architecture.md` §5 |
| `client_secret` hardcoded in `auth_oidc/data/sensorglobal_sso.xml` | `2026-04-21-odoo-integrations-analysis.md` §3 |
| Long-lived AWS access keys in `sensor-prod` secret (instead of EC2 role) | `2026-04-10-migration-plan.md` INV-08 |
| `.env` printed verbatim to CodeBuild logs on every build | `infra/deployment-pipeline.md` |

**Need from owners**: Which of these are known/accepted, which are oversights, who owns the remediation roadmap.

**Likely owner**: Platform Security / App Engineering.

---

## 13. Infrastructure provenance — ClickOps history

Almost the entire AWS environment was originally provisioned manually via the AWS Console (RDS parameter groups, ALB listener rules, SG ingress rules, EC2 UserData, EMQX config, Kafka config, CodeBuild buildspec, ...). Terraform state now exists from later import work, but it does not provide original change history or guarantee that all hidden runtime/configuration intent is captured.

| Need | Detail |
|---|---|
| **Change history** | Any informal log of what was changed when, and by whom? |
| **Tribal knowledge transfer** | Sessions with the original devops staff to capture undocumented config |
| **Buildspec source** | CodeBuild buildspec lives in console only — can we get a current export? |

**Why**: Migration to the new account requires re-creating all this. Without provenance, every difference between old and new is a debugging exercise.

**Likely owner**: DevOps Lead (original operator).

**Source**: `infra/infrastructure-provisioning.md`.

---

# 🟡 Medium

## 14. Error / incident management workflow

| Need | Detail |
|---|---|
| **Existing runbooks** | How are alarm-pipeline failures (e.g. SMS drop), MQTT outages, broker issues handled today? |
| **On-call rotation** | Who gets paged, via what channel |
| **Alerting config** | What CloudWatch / NewRelic alarms exist; thresholds; routes |

**Why**: We have ad-hoc incident notes (e.g. SMS drop, certificate expiry) but no consolidated incident-management picture. Required before recommending changes.

**Likely owner**: Ops / SRE lead.

---

## 15. Stale / legacy data — on-purpose or forgotten?

| Item | Question |
|---|---|
| `tbl_sms_allowed_list` has only 2 Indian test numbers (Dec 2022) | Is the SMS path gated on this allowlist? If yes, all AU customers are blocked — can't be true. So why does the table exist? |
| `DB_MYSQL_PASSWPRD` typo (not `PASSWORD`) in production secret | Anyone aware? Plan to fix? |
| Launch template `prod-api-server-lt` at version 650 | Manual edits over years — any record of what versions changed? |
| Keychain ALB shared across prod/qa/dev/sandbox | Intentional shared-tenant design or technical debt? |
| 9 orphaned Lambda functions from decommissioned QA/Dev/UAT | Safe to delete, or are any still wired to something? |
| Zabbix monitoring still running | New Relic is being deprecated; confirm Zabbix is the chosen replacement, then decommission New Relic |

**Likely owner**: Platform Lead / mixed.

**Source**: assorted (see individual references in `investigations/`).

---

## 16. Agency impersonation / "login-as"

| Need | Detail |
|---|---|
| **Confirmation of feature existence** | No code path found for super-admin to impersonate an agency user |
| **Operational procedure** | If support staff currently switch to agency view, how? Manual JWT mint? Shared creds? |

**Why**: Critical for support workflows and for our investigation (we cannot view the agency portal as an actual customer would). Also a security/audit concern.

**Likely owner**: Product owner + Support lead.

---

## 17. Property audit history

| Need | Detail |
|---|---|
| **Schema and retention of `tbl_property_history`** | What events are captured, how long retained |
| **Surfacing in UI** | Is there an admin view of audit history? Should there be? |
| **Compliance scope** | Does audit history feed any compliance/regulatory reporting? |

**Why**: Audit completeness affects compliance posture (Vanta) and tenancy-law obligations.

**Likely owner**: Compliance lead + Product owner.

---

## 18. Tenant legality / compliance

| Need | Detail |
|---|---|
| **State-by-state tenancy law mapping** | Which AU jurisdictions are supported and how compliance is enforced in code |
| **Data-handling for tenant PII** | Retention, redaction, right-to-erasure handling |

**Why**: Domain model captures the *data*; the *legal obligations* surrounding tenant records are not visible in code.

**Likely owner**: **Kristyn** (already flagged in original notes).

---

## 19. Lease management — operational ownership

| Need | Detail |
|---|---|
| **Who creates leases** in production today (agency staff vs imports)? |
| **Reconciliation with PropertyMe / PropertyTree / ConsoleCloud** | When tenants change at upstream PMS, how does it flow into our DB? |

**Why**: Lease lifecycle diagram is mechanical; operational reality (who does what, frequency, error modes) is unclear.

**Likely owner**: Operations lead + Integrations engineer.

---

## 20. Customer-facing surfaces (legal, support)

| Need | Detail |
|---|---|
| **Support email / phone** | Where customer issues land today. Who triages? |
| **Status page** | Does one exist? URL? Who updates during incidents? |
| **Terms & Conditions** version & owner | Current version, last review, who can amend |
| **Privacy policy** version & owner | Same |
| **Data deletion / right-to-erasure procedure** | Especially for tenants, given the platform retains PII |

**Likely owner**: Product Owner / Legal / Customer Success.

---

# 🟢 Low

## 21. Per-contact notification routing

| Need | Detail |
|---|---|
| **Product decision** | Current model routes by *role* only (agent/agency/tenants/landlord/contractor). Is per-contact routing a desired feature? |
| **Existing customer requests** | Has this been raised before? Any workaround currently in place? |

**Why**: Identified as a gap during Config-tab walkthrough. Needs a product call before any design work.

**Likely owner**: Product owner (Kristyn?).

---

# Suggested Next Step

Group all gaps into a single ask, batched by likely owner. Tier indicates priority for the conversation:

| Owner type | Critical | High | Medium | Low |
|---|---|---|---|---|
| **Hardware/firmware contact** | | 6 | | |
| **Platform/DevOps lead (original operator)** | 1, 2, 3 | 7, 8, 11, 13 | 15 | |
| **Product Owner** (Kristyn et al.) | 4 | | 16, 17, 18, 20 | 21 |
| **Mobile lead** | | 9 | | |
| **Finance / Accounts** | 4 | | | |
| **Platform Security / App Engineering** | 1 | 10, 12 | | |
| **Ops / SRE / Integrations** | 1 | | 14, 19 | |
| **Engineering Lead + Operations** | 5 | | | |

Once user reviews, items they can answer themselves will be removed; the remainder forms the email/meeting agenda for previous owners.
