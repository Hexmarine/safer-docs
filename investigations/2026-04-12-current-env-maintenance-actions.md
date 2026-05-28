# Current Environment — Maintenance Actions Required

**Date:** 2026-04-12  
**Scope:** Keep the existing production environment (account `747293622182`, `ap-southeast-2`) functioning through the 1–2 month migration window  
**Posture:** These are actions on the *source* account only — not migration tasks

---

## P0 — Broken Now

### 1. ACM wildcard cert expired — web apps serving broken HTTPS

**What:** DigiCert `*.sensorglobal.com` cert expired 2026-04-09. All production, UAT, QA, sandbox, and dev web app domains are presenting an expired certificate to browsers. `wordpress-prod-alb` and `sso-api-lb` are also affected.

**Impact:** Every end-user and developer visiting any `*.sensorglobal.com` portal sees a browser security warning. HTTPS is technically still working (not broken at the transport layer) but browsers will block or warn, and any client enforcing cert validity will fail.

**Fix:** Create two Amazon-managed ACM certs (DNS-validated via Route 53 one-click — Route 53 zone is in this account):
1. `us-east-1`: `*.sensorglobal.com` + `sensorglobal.com` → update all 4 CloudFront distributions
2. `ap-southeast-2`: `*.sensorglobal.com` + `sensorglobal.com` → set as default cert on all ALBs

Zero downtime. No cost. Auto-renews forever. ~15 minutes of console work.

**References:** `investigations/2026-04-12-certificate-expiry-incident.md`

---

### 2. SMS/alarm notifications down 93% since March 26 — active safety risk

**What:** Daily SMS sends dropped from ~350–585/day to ~29/day on March 26, 2026 and have not recovered. Root cause: a backend code deployment stopped populating `tbl_alarm_alerts`, which is the table that drives the SMS notification pipeline. All alert types (DISCONNECT, BATTERY, ALERT, TAMPERED, RESET) dropped uniformly — the entire alert creation pipeline stopped mid-day March 26.

**Impact:** This is a smoke alarm monitoring platform. If alarm events are not generating SMS notifications to residents, the core safety promise of the product is broken. This is the highest-priority issue in the entire codebase.

**Fix requires app team:**
- Identify what changed in the March 26 deployment (migration script, code refactor)
- Restore the MQTT event → `tbl_alarm_alerts` insertion pipeline
- Verify daily SMS send rate returns to 350+ /day after fix
- Separately audit `tbl_communications_queue` (899 undelivered email rows since April 2025 — email delivery is also broken)

**References:** `investigations/2026-04-11-sms-drop-investigation.md`, `investigations/2026-04-12-mysql-sms-root-cause.md`

---

## P1 — Running But at Risk

### 3. EMQX TLS cert on port 18756 expired

**What:** The `*.sensorglobal.com` TLS certificate on the EMQX broker's custom port 18756 expired 2026-04-08. Port 18756 is the primary hub connection port — 16 of 21 connected clients use it.

**Impact:** Currently no visible impact because hub firmware does not validate certificate expiry (consistent with embedded cellular IoT firmware). Hubs are still connecting. However:
- Any client that does validate certs will fail
- It is a security risk (cert should be valid for TLS integrity)
- The new broker in the target account must start with a valid cert — this is a blocker for migration Phase 4

**Fix:** Deploy a new `*.sensorglobal.com` cert to the EMQX EC2 instance (`i-0b381a2bfdf65a329`). Options:
- Use the new Amazon-managed cert from action #1 (export from ACM is not possible for ALB use, but ACM Private CA or Let's Encrypt via `certbot` on the EC2 instance would work)
- Alternatively use Let's Encrypt: `certbot certonly --dns-route53 -d '*.sensorglobal.com'` on the EMQX instance
- Restart EMQX listener after cert rotation (EMQX 5.x supports hot reload: `emqx_ctl listeners restart ssl:customport`)

**References:** `investigations/2026-04-10-inv-results.md` §INV-01

---

### 4. Kafka log data stored in `/tmp` — lost on any reboot

**What:** The Kafka broker (`i-0b381a2bfdf65a329` co-hosted with EMQX, or separate — confirm) has `log.dirs=/tmp/kafka-logs`. The broker has been running since 2026-02-22 (7+ weeks) without reboot.

**Impact:** Any EC2 reboot, stop/start, or instance replacement wipes all Kafka topic data and consumer offsets. The `logs-production` topic holds 7 days of production device events (236 MB). A sudden reboot would cause:
- Loss of up to 7 days of unprocessed events
- Consumer group `smoke-alarm` loses its offset checkpoint and would re-start from `latest`, missing any events produced during the outage

**Fix:** Either:
- Accept the risk until migration (Kafka is not authoritative state — it's a streaming log). Kafka data is not being migrated anyway.
- Or: Mount an EBS volume (e.g. `/data/kafka-logs`), update `log.dirs`, restart Kafka. Requires a brief Kafka downtime (~1 min).

**Recommendation:** Accept the risk if a reboot is not planned. Make sure nobody stops/reboots this instance unexpectedly. Schedule the fix as part of the new-account Kafka setup (start clean on EBS-backed volume).

**References:** `investigations/2026-04-10-inv-results.md` §INV-08

---

### 5. Email delivery broken — 899 undelivered rows since April 2025

**What:** `tbl_communications_queue` has 899 rows all with `type='email'`, `status=0` (undelivered), spanning April 2025 to present. These are emails that were queued but never sent.

**Impact:** Unknown scope — depends on what this overnight queue delivers (likely tenant/operator notifications, not alarm alerts). One year of undelivered emails suggests a silent failure in the email delivery path (SendGrid API key rotation? queue processor stopped?).

**Fix requires app team:** Check the queue processor service, verify SendGrid API key is valid, investigate why `status` is not advancing from 0.

**References:** `investigations/2026-04-12-mysql-sms-root-cause.md`

---

## P2 — Housekeeping / Risk Mitigation

### 6. Domain AutoRenew — verify payment method before mid-2026

**What:** Several domains expire mid-2026 and are on AutoRenew. If the source account's payment method lapses or the account is closed before these renewals fire, domains will expire.

| Domain | Expiry window |
|--------|--------------|
| `sensorglobal.au`, `sensorglobal.net` | ~June 2026 |
| `sensorglobal.co.uk`, `sensorglobal.com.au` | ~July 2026 |
| `sensorinsure.com`, `sensorinsure.com.au` | ~September 2026 |

**Fix:** Verify a valid credit card is attached to the source account now. Do not close the source account before these domains are either renewed or transferred to the new account.

---

### 7. `OIDC_ISSUER=http://localhost:4100` — SSO misconfiguration

**What:** The SSO service (`sensor-prod-sso`) is configured with `OIDC_ISSUER=http://localhost:4100`. This hardcoded localhost URL is embedded in all issued JWTs as the `iss` claim. Any client that validates the issuer URL (e.g. via OIDC discovery) against the actual server URL will fail.

**Impact:** Currently not causing failures because clients likely skip issuer URL validation. Risk: if the SSO server is restarted on a different port, or if a security library is updated to enforce issuer validation, all active portal sessions break.

**Fix:** Update `OIDC_ISSUER` to the actual public-facing SSO URL (e.g. `https://keychain.sensorglobal.com` or similar) and coordinate a rolling deploy. All active JWTs will be invalidated — users will need to re-login.

**Note:** This is a migration blocker too — the new account must start with the correct issuer URL.

**References:** `investigations/2026-04-12-auth-architecture.md`

---

### 8. `tbl_sms_allowed_list` — audit required

**What:** A table `tbl_sms_allowed_list` exists with only 2 entries — both Indian test phone numbers from December 2022. If any production SMS code path gates on this allowlist, no Australian customer has ever received SMS through that path.

**Impact:** Unclear — SMS was working before March 26 (350+/day), suggesting the primary SMS path does not gate on this table. But if there is a secondary SMS path that does, it has been silently failing since 2022.

**Fix requires app team:** Search codebase for any query against `tbl_sms_allowed_list`. If it's used in production, either populate it correctly or remove the gate.

---

## Summary Table

| # | Issue | Priority | Owner | Effort |
|---|-------|----------|-------|--------|
| 1 | ACM wildcard cert expired — browsers blocked | **P0** | DevOps | ~15 min console |
| 2 | SMS/alarm notifications 93% drop since Mar 26 | **P0 Safety** | App team | Unknown — needs code investigation |
| 3 | EMQX TLS cert expired on port 18756 | **P1** | DevOps | ~30 min (certbot + restart) |
| 4 | Kafka logs in `/tmp` — lost on reboot | **P1** | DevOps | Accept risk OR 1h EBS mount |
| 5 | Email delivery broken — 899 queued undelivered | **P1** | App team | Investigate queue processor |
| 6 | Domain AutoRenew payment verification | **P2** | Finance/DevOps | 5 min check |
| 7 | `OIDC_ISSUER=localhost` misconfiguration | **P2** | App team | Config change + rolling deploy |
| 8 | `tbl_sms_allowed_list` audit | **P2** | App team | Code search |
