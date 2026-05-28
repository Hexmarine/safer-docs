# Gaps — Compact List

One-line-per-item summary of open gaps. Sorted by criticality. See [`gaps-detailed.md`](./gaps-detailed.md) for the full context, evidence and likely-owner tables.

Legend: 🔴 critical · 🟠 high · 🟡 medium · 🟢 low

---

## 🔴 Critical

- **1. Systems & accounts access**
  - **AWS** — ✅ have it; source/parent account will detach, ours to keep
  - **Domains** — ✅ all 14 in AWS Route 53 Registrar; transfer with AWS account migration (not external registrar). Just verify payment method on source account before mid-2026 renewals.
  - **EMQX broker** — read-only `emqx_ctl` + dashboard
  - **Databases** — MySQL replica, MongoDB Atlas read user, anonymised dump
  - **Mobile** — Apple Developer, Google Play, signing keys (iOS profiles, Android upload key), APNs/FCM
  - **Firmware** — 3rd-party-developed for SensorGlobal (vendor TBD); need source repo, build pipeline, OTA flow, vendor contact
  - **Sandbox/soapbox** — creds + seed data + test SIM/hub + test app build
  - **Comms** — Twilio, SendGrid, KORE SIM
  - **Observability** — Sentry, LaunchDarkly, Bitrise, Firebase/GCP _(New Relic — deprecating, skip)_
  - **Email-domain** — DMARC/SPF/DKIM owner; ACM auto-renewal monitoring
  - **Business systems** (see breakdown below)

### Business systems — purpose & integration

| System | Purpose | Integration | Owned by |
|---|---|---|---|
| **Odoo** (self-hosted EC2) | Financial backend: invoicing, subscriptions, agency CRM, credit applications, payment processing, support tickets | OAuth basic-auth token (`ODOO_BASICAUTH_USERNAME/PASSWORD`, `ODOO_END_URL`); backend ↔ Odoo bidirectional. Backend pulls invoices/PDFs/subscriptions, pushes property status / contractor signup / credit app / agency registration / ticket creation; Odoo webhooks back into backend (property status, cart status, contractor data). Daily reconciliation cron. `tbl_odoo_api_keys`, `properties.odoo_status`, `*.odoo_invoice_id`. Custom addons in `code/odoo-addons/`: `sg_rest_api`, `sg_payment_cba`, `sg_payment_credit`, `sg_property_agency`, `sg_crm`, `sg_util`, `sg_website`, `auth_oidc`, `shippit_odoo_integration`, `xf_excel_odoo_connector` | us / DevOps |
| **PropertyMe** | Agency CRM — import properties/agents/contacts from real-estate agencies | OAuth authorization-code flow (`PROPERTYME_API_LOGIN_URL`, `_CLIENT_ID`, `_SECRET`, `_CALLBACK_URL`, `_TOKEN_URL`, `_BASE_URL`, `_FRONTEND_URL`). Per-tenant tokens stored in `tbl_propertyme_api_tokens` | PropertyMe AU |
| **PropertyTree** | Agency CRM (alternative) | REST with `PROPERTYTREE_BASE_URL` + `PROPERTYTREE_SUBSCRIPTION_KEY` + `PROPERTYTREE_APPLICATION_KEY` (`/apikey/v1/application_keys/...`); per-agency keys in `tbl_propertytree_api_keys` | MRI Software |
| **Console Cloud** | Agency CRM (alternative) | OAuth client-creds + per-tenant refresh token + JWKS verification + webhook ingress (`CONSOLE_CLOUD_API_URL`, `_CLIENT_ID`, `_SECRET`, `_APP_PARTNER_CODE`, `_WEBHOOK_URL`, `_VALID_IPS`); `tbl_console_cloud_api_tokens` | Console Group |
| **NEXU** | External property data enrichment / registry lookup | Bearer-token REST: `GET ${NEXU_API_URL}/properties/{id}` | NEXU |
| **Corpsure** | Landlord-insurance broker — generates insurance reports for properties | Login API (`CORPSURE_LOGIN_API` + username/password), insurance-data URL (`CORPSURE_ISURANCE_DATA_URL`), terms URL; reports persisted to S3 (`corpsure/{id}`) and `tbl_corpsure_reports*`. Manual feature flag `MANUAL_CORPSURE_API` | Corpsure |
| **CBA** (Commonwealth Bank) | Payment provider for invoices | **Odoo-only** — addon `sg_payment_cba`; backend never calls CBA directly. Card-on-file / authorization / payment IDs handled via Odoo `payment.transaction` | CBA + Odoo |
| **Shippit** | Shipping carrier — dispatches hubs/alarms to contractors & customers | **Odoo-only** — addon `shippit_odoo_integration` (delivery_carrier, package types, rate/ship endpoints under `Shippit_api_url`); backend never calls Shippit directly | Shippit |

Net: **Odoo is the integration hub for finance + CBA + Shippit + agency mass-mailing.** Backend talks to Odoo, PropertyMe, PropertyTree, Console Cloud, NEXU, Corpsure directly.

- **2. Active P0/P1 production incidents** — owner needed
  - SMS pipeline 93% drop since 2026-03-26 (smoke-alarm safety)
  - 899 emails stuck in `tbl_communications_queue` since April 2025
  - EMQX TLS cert expired on port 18756 (16 of 21 hubs)
  - DLM Kafka snapshot tag mismatch — no Kafka EBS snapshots since Nov 2024
- **3. Backup & DR coverage gaps**
  - MongoDB Atlas backup policy not verified
  - Odoo filestore (75 GB) — no portable export
  - EMQX auth/ACL/retained/certs — no `emqx_ctl` export automation
  - Terraform state S3 not cross-region replicated
  - Kafka data on `/tmp` — lost on reboot
  - Redis snapshot retention only 1 day

## 🟠 High

- **4. Appinventiv access — REVOKE (decided).** The JV has **separated from Appinventiv** (no ongoing relationship; Safer Homes now does maintenance/dev/support). All Appinventiv access must be removed: 6 IAM users still active; `lokendra` personal-IP allowlist (ALL traffic from `116.73.36.32/32`); the `sensorglobal.msp@appinventiv.com` prod on-call path is **defunct — do not use**. Action: revoke per `runbooks/12-aws-access-containment-follow-up.md`.
- **5. Auth/security debt — intent vs accident**
  - `OIDC_ISSUER=http://localhost:4100`
  - Hardcoded `SALT=SMOKE@@123`
  - Plaintext OTPs, no expiry
  - JWT blocklist no TTL
  - 989k stale `isLogin=1` mobile sessions
  - 2FA enabled for only 7 of 56,133 users — **confirmed control gap**: the Information Security Overview commits to *mandatory* 2FA/MFA for customer-data and infrastructure access, so this is policy-vs-reality, not just a stat
  - `client_secret` hardcoded in `auth_oidc/data/sensorglobal_sso.xml`
  - Long-lived AWS access keys in `sensor-prod` secret
  - `.env` printed to CodeBuild logs every build
- **6. Infrastructure provenance (ClickOps)** — change history, tribal knowledge transfer, CodeBuild buildspec export

## 🟡 Medium

- **7. Error / incident management workflow** — runbooks, on-call rotation, alerting routes
- **8. Stale / legacy data — on purpose or forgotten?**
  - `tbl_sms_allowed_list` has 2 Indian test numbers (Dec 2022)
  - `DB_MYSQL_PASSWPRD` typo in prod secret
  - Launch template `prod-api-server-lt` at version 650
  - Keychain ALB shared across prod/qa/dev/sandbox
  - 9 orphaned Lambdas from removed QA/Dev/UAT
  - Zabbix running alongside New Relic
- **9. Agency impersonation / "login-as"** — does it exist? If not, how does support reproduce customer issues?
- **10. Property audit history** — `tbl_property_history` schema, retention, UI surfacing, compliance scope
- **11. Tenant legality / compliance** — AU state-by-state tenancy mapping, PII retention, right-to-erasure (Kristyn)
- **12. Lease management — operational ownership** — who creates leases in prod; PropertyMe/PropertyTree/ConsoleCloud reconciliation
- **13. Customer-facing surfaces** — support email/phone, status page, T&Cs version/owner, privacy policy version/owner, deletion procedure

## 🟢 Low

- **14. Per-contact notification routing** — product decision: extend role-based routing to specific contacts?
