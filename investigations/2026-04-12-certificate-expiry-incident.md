# Certificate Expiry Incident — sensorglobal.com

**Date:** 2026-04-12  
**Severity:** CRITICAL (active production impact)  
**Status:** RESOLVED — 2026-04-13

---

## Summary

The DigiCert wildcard certificate for `*.sensorglobal.com` (covering both the wildcard and bare domain) that was imported into ACM expired on **2026-04-09** (3 days ago). This cert was the sole TLS cert on all four CloudFront distributions that serve the production web apps, as well as several ALBs. All user-facing HTTPS endpoints on `sensorglobal.com` subdomains are currently presenting an **expired certificate** to browsers and clients.

There is exactly one valid ACM cert in the account: an Amazon-managed cert for `*.sensorglobal.com` (no bare domain) in `ap-southeast-2`, expiring 2026-06-23. It is not usable by CloudFront (which requires certs in `us-east-1`) and has no bare-domain SAN.

---

## ACM Certificate Inventory

### ap-southeast-2

| Domain | Issuer | Status | Expires | In Use By |
|--------|--------|--------|---------|-----------|
| `*.sensorglobal.com` | Amazon | **ISSUED ✅** | 2026-06-23 | 6 ALBs (as non-default SNI cert) |
| `*.sensorglobal.com` + bare | DigiCert | **EXPIRED ❌** | 2026-04-09 | 14 ALBs (default cert on most) |
| `*.sensorglobal.com` + bare | DigiCert | **EXPIRED ❌** | 2025-05-12 | 13 ALBs |
| `*.sensorglobal.com` + bare | DigiCert | **EXPIRED ❌** | 2024-05-12 | 11 ALBs |
| `*.sensorglobal.com` + bare | DigiCert | **EXPIRED ❌** | 2023-05-12 | 9 ALBs |
| `server` | — | **EXPIRED ❌** | — | — |
| `client1.domain.tld` | — | **EXPIRED ❌** | — | — |
| `sensorinsure.com.au` | — | **EXPIRED ❌** | — | — |
| `sensorinsure.com` | Amazon | **ISSUED ✅** | — | — |
| `sensorinsure.com` | Amazon | **ISSUED ✅** | — | — |

### us-east-1 (used by CloudFront)

| Domain | Issuer | Status | Expires |
|--------|--------|--------|---------|
| `*.sensorglobal.com` | DigiCert | **EXPIRED ❌** | 2026-04-09 |
| `*.sensorglobal.com` | DigiCert | **EXPIRED ❌** | earlier |
| `*.sensorglobal.com` | DigiCert | **EXPIRED ❌** | earlier |
| `*.sensorglobal.com` | DigiCert | **EXPIRED ❌** | earlier |
| `*.sensorglobal.com` | DigiCert | **EXPIRED ❌** | earlier |

**There are zero valid certs in us-east-1.** CloudFront requires certs in us-east-1.

---

## Affected Services

### CloudFront (CRITICAL — all production web apps)

All 4 named distributions use the expired cert `dbd399a5` (us-east-1, expired 2026-04-09):

| Distribution | Affected Domains |
|---|---|
| Production | `admin.sensorglobal.com`, `trader.sensorglobal.com`, `agent.sensorglobal.com`, `owner.sensorglobal.com`, `activation.sensorglobal.com`, `reschedule.sensorglobal.com` |
| UAT | `admin-uat.sensorglobal.com`, `trader-uat.sensorglobal.com`, `agent-uat.sensorglobal.com`, `owner-uat.sensorglobal.com`, `activation-uat.sensorglobal.com`, `reschedule-uat.sensorglobal.com` |
| QA | `admin-qa.sensorglobal.com`, `trader-qa.sensorglobal.com`, `agent-qa.sensorglobal.com`, `owner-qa.sensorglobal.com`, `activation-qa.sensorglobal.com`, `reschedule-qa.sensorglobal.com` |
| Sandbox | `admin-sandbox.sensorglobal.com`, `agent-sandbox.sensorglobal.com`, `trader-sandbox.sensorglobal.com`, etc. |
| Development | `admin-development.sensorglobal.com`, `trader-development.sensorglobal.com`, etc. |

### ALBs — expired cert as only cert (no fallback)

These ALBs have ONLY the expired DigiCert cert attached:

| ALB | Account | Impact |
|-----|---------|--------|
| `wordpress-prod-alb` | 747293622182 | Production WordPress site |
| `sso-api-lb` | 747293622182 | SSO API (production) |
| `qa-sso-api` | 747293622182 | QA SSO API |
| `sandbox-sso-api` | 747293622182 | Sandbox SSO API |
| `smoke-uat-api` | 747293622182 | UAT smoke/API |
| `uat-sso-api-alb` | 747293622182 | UAT SSO ALB |
| `wordpress-qa` | 747293622182 | QA WordPress |
| `prod-syd-1-az1-1-0` | **813668149731** | Production (external account) |
| `prod-syd-1-az2-1-26` | **813668149731** | Production (external account) |
| `prod-syd-1-az3-1-3` | **813668149731** | Production (external account) |

### ALBs — valid cert present but expired cert is the DEFAULT

These ALBs have the valid Amazon cert (`ae4603fd`) plus the expired cert as the listener default. AWS ALB selects by SNI, so `*.sensorglobal.com` requests served correctly; bare-domain or no-SNI requests fall back to the expired default.

- `Smoke-Sandbox-Api-LB`, `SmokeAPI-LB`, `SmokedevelopmentLB`, `SmokeqaLB`
- `keychain-production-ALB`, `sso-dev-api-lb`

---

## Root Cause

The pattern of cert management in this account has been:
1. Purchase DigiCert wildcard cert annually
2. Import it into ACM in both ap-southeast-2 and us-east-1
3. Attach manually to all ALBs and CloudFront distributions
4. No automated renewal path — INELIGIBLE for ACM managed renewal (imported certs)

The 2026 DigiCert cert expired 2026-04-09. The valid Amazon-issued cert (`ae4603fd`) was created in May 2025 but only in ap-southeast-2, only for the subdomain wildcard (no bare domain), and was never added to CloudFront.

Accumulation of 4 generations of expired imported certs on ALBs creates confusion but is not harmful in itself (ALBs do not serve expired certs when a valid one is available via SNI).

---

## Remediation — Required Actions

> These are recommendations only. Each requires explicit approval before execution.

### Immediate (P0 — production down)

1. **Create Amazon-managed cert in us-east-1** for `*.sensorglobal.com` + `sensorglobal.com` (DNS validation via Route 53). This will auto-renew.
2. **Update all 4 CloudFront distributions** to use the new us-east-1 cert.
3. **Update ALBs with only expired cert** (wordpress-prod-alb, sso-api-lb) to use the ap-southeast-2 valid cert `ae4603fd` as a stop-gap, then replace the default cert once a proper cert covering both wildcard + bare domain is available.

### Short-term (P1)

4. **Create new Amazon-managed cert in ap-southeast-2** covering both `*.sensorglobal.com` and `sensorglobal.com` (DNS validation). Replace the current Amazon cert which lacks the bare domain.
5. **Update all ALB default certs** to the new ap-southeast-2 cert.
6. For the 3 ALBs in account `813668149731`, coordinate cert rotation with that account's owner.

### Cleanup (P2)

7. Remove all 4 expired DigiCert certs from ALB listener cert lists.
8. Delete expired ACM certs from both regions.

---

## Notes on the Valid Cert (ae4603fd)

- ARN: `arn:aws:acm:ap-southeast-2:747293622182:certificate/ae4603fd-a592-4193-a7f4-69a307cc8be3`
- Domain: `*.sensorglobal.com` only (no bare `sensorglobal.com`)
- Expires: 2026-06-23 (~72 days from today)
- Renewal eligibility: ELIGIBLE (Amazon-managed, DNS validated)
- This cert **cannot** be used by CloudFront (wrong region) and has no bare-domain SAN
- It is currently the non-default cert on 6 ALBs — those ALBs are partially protected for wildcard subdomains only

---

## Investigation Queries

```bash
# List all ACM certs, both regions
AWS_PROFILE=sensorsyn-mfa aws acm list-certificates --region ap-southeast-2 --output table
AWS_PROFILE=sensorsyn-mfa aws acm list-certificates --region us-east-1 --output table

# Check CloudFront cert usage
AWS_PROFILE=sensorsyn-mfa aws cloudfront list-distributions \
  --query 'DistributionList.Items[*].{Aliases:Aliases.Items,Cert:ViewerCertificate.ACMCertificateArn}' \
  --output json
```

---

## Resolution — 2026-04-13

All expired DigiCert certificates replaced with Amazon-issued ACM certificates.

### New certs issued

| Region | ARN (short) | Expiry | Auto-renew |
|--------|-------------|--------|------------|
| us-east-1 | `5dbe04dc-dee3-4021-a004-2ab8dbcfe750` | Oct 27 2026 | yes |
| ap-southeast-2 | `d41745f4-1816-464d-985e-9251ed25ae04` | Oct 27 2026 | yes |

DNS validation used the pre-existing wildcard CNAME in Route 53 — no DNS changes required. Both certs issued in under 30 seconds.

### Changes applied

- **5 CloudFront distributions** updated to `5dbe04dc` (us-east-1) — all propagated to Deployed
- **13 ALBs** default cert updated to `d41745f4` (ap-southeast-2)
- **4 expired DigiCert certs removed** from all ALB listener SNI lists:
  - `2f12b37b` (expired Apr 9 2026)
  - `a545b236` (expired May 2025)
  - `a750eea7` (expired May 2023)
  - `38994958` (expired May 2024)

### Endpoint verification (2026-04-13 07:46 AEST)

All 16 HTTPS endpoints confirmed serving Amazon RSA cert, expiry Oct 27 2026:

| Environment | Endpoints | TLS | HTTP |
|-------------|-----------|-----|------|
| Production | admin, trader, agent, owner, activation, reschedule, web | Oct 2026 | 200 |
| UAT | admin-uat, trader-uat | Oct 2026 | 200 |
| QA | admin-qa, trader-qa, web-qa | Oct 2026 | 403 (WAF) |
| Sandbox | admin-sandbox, trader-sandbox | Oct 2026 | 200 |
| Dev | admin-development, trader-development | Oct 2026 | 403 (WAF) |

403 responses on QA/Dev are pre-existing WAF/IP restrictions, unrelated to certs.

### Still deferred

- 3 ALBs in external account `813668149731` — separate plan required
- `ae4603fd` (wildcard-only, valid Jun 2026) — still in some listener SNI lists; retire before Jun 2026
- ACM cleanup: delete the 4 expired DigiCert certs from ACM console
