# Odoo Integrations Analysis

**Date**: 2026-04-21  
**Status**: Investigation  
**Scope**: All custom Odoo addons, REST API surface, backend↔Odoo data flows, payment/shipping/SSO integrations

---

## Executive Summary

SensorGlobal uses **Odoo 16 Enterprise** (PostgreSQL-backed, Docker-deployed) as its core CRM and internal operations system. It sits alongside the Node/Express backend — not replacing it — handling billing, subscriptions, property lifecycle, CRM, and e-commerce.

Integration pattern is primarily **unidirectional**: the Node backend syncs operational state into Odoo via a custom REST API. Odoo then handles invoicing, credit management, and customer comms. Payment (CBA), shipping (Shippit), and SSO (custom OIDC) are all Odoo-native integrations.

**14 custom addons** are maintained in `code/odoo-addons/`.

---

## Architecture Overview

```
FRONTEND LAYER (Angular + Mobile)
├── admin.sensorglobal.com (CloudFront + S3)
├── agent / owner / trader / activation portals
└── iOS/Android apps

API GATEWAY LAYER
├── api.sensorglobal.com  → Node/Express backend (EC2)
└── auth.sensorglobal.com → SSO provider (OIDC)

APPLICATION LAYER
├── Node/Express Backend (sensor-alarm-backend)
│   └── Device mgmt, jobs, properties, subscriptions → MySQL + MongoDB
├── SSO Provider (sso-provider)
│   └── OpenID Connect / OAuth2
└── Odoo 16 (CRM + Operations)
    └── PostgreSQL

EXTERNAL INTEGRATIONS (Odoo-initiated)
├── PropertyMe     — property management sync (bidirectional)
├── Shippit        — shipping rate quotes + label generation
├── CBA/Simplify   — payment gateway
├── SendGrid       — mail relay via Postfix
└── Wild Dog Solutions — agency/landlord data

DATA LAYER
├── MySQL RDS       — operational state (backend)
├── MongoDB         — event logs, audit trails
├── Redis           — sessions, caching
├── PostgreSQL      — Odoo
└── S3              — media, documents
```

**Database environments** (auto-routed by hostname):
| Hostname | Database |
|---|---|
| crm.sensorglobal.com | sensorglobal_live |
| crm-qa.sensorglobal.com | sensorglobal_qa |
| crm-sandbox.sensorglobal.com | sensorglobal_sandbox |
| crm-development.sensorglobal.com | sensorglobal_dev |

---

## Custom Addons

### 1. `rest_api` — Core REST Framework
- **Author**: portcities (v16.0.1.0.0)
- **Purpose**: Infrastructure layer enabling external REST access to Odoo models.
- **Endpoints**:
  | Endpoint | Method | Auth |
  |---|---|---|
  | `/api/<version>/oauth/token` | POST | Basic Auth (client_id:secret) |
  | `/api/<version>/<model>/<method>` | JSON-RPC | Bearer Token |
  | `/api/<version>/media/<model>/<method>/<id>` | HTTP | Bearer Token |
- **Token format**: HMAC-SHA1 signature + timestamp + UID, 3600s TTL
- **Must be in** `server_wide_modules` at startup

### 2. `sg_rest_api` — SensorGlobal REST API
- **Author**: Portcities (v15.0.1.0.0)
- **Purpose**: Platform-specific endpoints exposing CRM, billing, and operational data
- **Depends on**: rest_api, sg_property_agency, sg_util, account, sale_subscription, website_sale
- **Models exposed**:
  | Model | Key Method |
  |---|---|
  | account.move | `v1_get_all_invoice` — fetch invoices by sensor_live_id |
  | sale.order | `v1_get_all_subscription` — paginated subscriptions |
  | crm.lead | Lead/opportunity sync |
  | helpdesk.ticket | Support ticket data |
  | res.partner | Customer details |
  | product.template | Product catalog |
  | account.pricelist | Pricing tiers |
  | ir.attachment | Documents/media |
- **Cross-system key**: `sensor_live_id` links Odoo partners to backend properties

### 3. `auth_oidc` — SSO / OpenID Connect
- **Author**: ICTSTUDIO / ACSONE (v16.0.1.0.0)
- **Purpose**: Odoo users log in via the platform's OIDC provider
- **Flow**:
  1. Odoo login → "Log In with SensorGlobal SSO"
  2. Redirect to auth service (`auth.sensorglobal.com`)
  3. JWT returned, validated against `/jwks` endpoint
  4. Odoo user created/linked by email
- **Config** (in `data/sensorglobal_sso.xml`):
  - `client_id`: `sensor_odoo_dev_client`
  - Auth endpoint: `api.sensorglobal.com/api/v1/users/sso/loginWithSSO`
  - Token endpoint: `auth.sensorglobal.com/token`
  - JWKS: `auth.sensorglobal.com/jwks`
  - User profile: `api.sensorglobal.com/api/v1/users/profile?validateSSO=true`
- ⚠️ **client_secret hardcoded in XML** (committed to repo)

### 4. `sg_payment_cba` — CBA / Simplify Commerce Payments
- **Author**: Sensor Global Pty Ltd (v16.0.1.0.0)
- **Category**: Accounting/Payment Providers (code=`cba`)
- **Depends on**: Simplify Commerce Python SDK (`simplifycommerce-sdk-python`)
- **Flow**:
  1. Customer places order → `payment.transaction` created
  2. Frontend JS calls Simplify Commerce API → returns `paymentId` + `authCode`
  3. Backend POSTs to `/payment/validate`
  4. Odoo calls `simplify.Payment.find(paymentId)` to verify
  5. Transaction set to APPROVED/ERROR
  6. Invoice email triggered (`sg_website.email_template_O14`)
- **Keys stored in**: `payment.provider` fields (`cba_public_key`, `cba_private_key`)

### 5. `sg_payment_credit` — In-House Credit Accounts
- **Author**: Sensor Global (v16.0.1.0.0)
- **Purpose**: Credit account system with approval workflow and dunning
- **Key fields** on `res.partner`:
  - `sg_credit_approved` — approved for credit
  - `credit_account_blocked` — auto-set during dunning window
- **Dunning schedule** (monthly cron):
  - 15th: 1st reminder
  - 16th: 2nd reminder
  - 17th: 3rd reminder → account blocked if unpaid
  - 4 email templates (O01, credit_1–4)
- **MySQL side**: `tbl_credit_application` / `tbl_credit_application_detail`

### 6. `shippit_odoo_integration` — Shipping
- **Author**: Vraja Technologies (v16.0.15.02.2023)
- **Purpose**: Real-time shipping rate quotes + label generation via Shippit API
- **API**: `GET /api/3/quotes?auth_token=<token>`
- **Config**: Shippit API URL + token stored on `res.company`
- **Flow**: Quote request per sale order → rates stored in `shippit_shipping_charge_ids`

### 7. `sg_crm` — CRM Customisations
- **Version**: 15.0.1.0.0
- **Purpose**: Property management–specific CRM fields
- **Adds to**: `crm.lead`, `res.partner`, `res.config.settings`
- **Custom data**: Contact stages, investor types, campaign associations, industry classifications, postcode→property mapping

### 8. `sg_property_agency` — Property & Subscription Lifecycle
- **Version**: 15.0.1.0.0
- **Purpose**: Core property lifecycle, subscriptions, agency management
- **Depends on**: sale_subscription, queue_job, base_geolocalize
- **Key models**: crm.lead (extended), res.partner (extended), sale.order, account.move, `sync_log` (PropertyMe audit)
- **Includes**: Email templates for onboarding, password reset, etc.

### 9. `sg_util` — Shared Utilities
- **Version**: 16.0.0.0.1
- **Purpose**: Common helpers, mail templates, credit tracking, aged receivables
- **Templates**: O01–O20 series + SUP01 (business workflow emails)

### 10. `sg_website` — Customer Portal / E-Commerce
- **Version**: 16.0.1.0.0
- **Depends on**: website_sale, auth_oidc, sg_property_agency
- **Features**: Product catalogue, e-commerce storefront, landing pages (A/B), helpdesk portal, property owner/tenant/contractor views

### 11. `xf_excel_odoo_connector` — BI / Excel Export
- **Author**: XFanis (third-party)
- **Purpose**: Export Odoo data to Excel (.odc) / Power BI for business intelligence

### 12–14. Community addons (bundled)
- **queue_job** (v16.0.2.3.1) — Async job queue for property syncs
- **partner_firstname** (v16.0.1.0.1) — Split first/last name on contacts
- **mass_mailing_partner** (v16.0.1.0.0) — Link partners to mailing lists

---

## Backend ↔ Odoo Data Flow

| Backend Event | Odoo Action | Purpose |
|---|---|---|
| Property created/updated | `sale.order` → subscription record | Subscription sync |
| Subscription activated | `res.partner.sensor_live_id` updated | Link customer to backend |
| Invoice issued | `account.move` query via REST | Fetch invoice for UI |
| Payment received | `account.move.state` → posted | Accounting sync |
| Support ticket created | `helpdesk.ticket` created | CRM agent visibility |
| Dunning cycle | `res.partner.credit_account_blocked` | Block new orders |

**Auth**: Backend stores Odoo API credentials in MySQL `tbl_odoo_api_keys`, calls `/api/v1/oauth/token` for Bearer tokens, includes token in subsequent REST calls.

**Config keys** (backend env vars): `ODOO_END_URL`, `PROPERTY_OWNER_URL`

---

## Deployment

**Stack**: Docker Compose on EC2 (`code/odoo-configuration/`)  
**Services**: PostgreSQL 14 (bitnami), Postfix SMTP relay → SendGrid, Odoo 16 Enterprise, Caddy (reverse proxy + TLS)  
**Mail relay**: Postfix rewrites `FROM` header and forwards to SendGrid via SMTP  
**Odoo startup modules**: `base,web,queue_job,rest_api` (must be in `server_wide_modules`)  
**Workers**: 2 Odoo workers, 1 cron thread (intentionally limited)  
**Addons path**: `s/addons, e/enterprise, e/sensorglobal, e/odoo-nonproductionmode, e/server-auth`

---

## Integration Summary

| System | Direction | Protocol | Auth | Risk |
|---|---|---|---|---|
| Backend → Odoo | Outbound | REST/JSON | Bearer Token | Medium |
| Odoo → SSO | Outbound | OIDC auth code | OAuth2 | Low |
| Odoo ↔ Shippit | Bidirectional | REST/JSON | API Token | Medium |
| Odoo ↔ CBA | Outbound | SDK (HTTPS) | API Keys | High |
| Odoo ↔ SendGrid | Outbound | SMTP relay | Password | Low |
| Odoo ↔ PropertyMe | Bidirectional | REST/JSON | Token | Medium |

---

## Risks & Observations

### Risks
1. **Hardcoded secrets**: SSO `client_secret` committed in `auth_oidc/data/sensorglobal_sso.xml` — should be rotated and moved to AWS Secrets Manager
2. **No API rate limiting**: REST API framework has no throttling
3. **No token refresh**: Bearer tokens expire after 1h; no documented refresh flow
4. **Tight addon coupling**: `sg_rest_api` depends on 6 other addons — Odoo version upgrades are risky
5. **Dunning cron**: No retry mechanism if cron fails; customers could be missed
6. **DB hostname routing**: Misconfiguration could route requests to wrong environment's database

### Gaps
- No OpenAPI/Swagger spec for REST API surface
- No generic audit trail for backend↔Odoo sync failures
- No explicit subscription pause/cancel flow exposed via REST API
- REST API permission model is opaque (token-based, no RBAC documented)

### Positive Observations
- `sensor_live_id` acts as clean cross-system tenant key
- Good separation: operational state in MySQL, business records in Odoo
- REST API is stateless — horizontally scalable
- Auth (OIDC), payments (CBA), shipping (Shippit) all cleanly isolated in own addons
