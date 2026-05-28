# SMS Drop Investigation — P1 Safety Incident

**Date:** 2026-04-11  
**Account:** `747293622182` (ap-southeast-2)  
**Classification:** 🚨 P1 — Production Safety Concern  
**Status:** Root cause mechanism confirmed via live DB (2026-04-12) — proximate trigger (code deployment vs data change) requires app team confirmation. See `2026-04-12-mysql-sms-root-cause.md`.

---

## Summary

Daily SMS sends dropped from **350–585/day** (March 20–25) to **29/day** (March 27), an **~93% reduction**. The drop occurred on or around **March 26–27, 2026** and has persisted through April 11. For a smoke alarm monitoring platform, this represents a potential failure to alert residents of fire events.

---

## Evidence

### Daily SMS sent/delivered counts (AWS/SMSVoice CloudWatch)

| Date | Sent | Delivered | Delivery Rate |
|---|---|---|---|
| Mar 20 | 585 | 561 | 95.9% |
| Mar 21 | 452 | 439 | 97.1% |
| Mar 22 | 352 | 322 | 91.5% |
| Mar 23 | 342 | 324 | 94.7% |
| Mar 24 | 469 | 438 | 93.4% |
| Mar 25 | 381 | 345 | 90.6% |
| **Mar 26** | **201** | **193** | 96.0% |
| **Mar 27** | **29** | **29** | **100%** |
| Mar 28 | 31 | 31 | 100% |
| Mar 29 | 70 | 70 | 100% |
| Mar 30 | 70 | 70 | 100% |
| Mar 31 | 41 | 40 | 97.6% |
| Apr 1 | 50 | 47 | 94.0% |
| Apr 2–10 | 20–105 | 20–103 | ~97–100% |

### Monthly AWS SNS SMS spend (SMSMonthToDateSpentUSD)

| Period | Daily spend rate |
|---|---|
| March 1–25 | **~$15–18/day** |
| March 26–31 | **~$1–3/day** |
| April 1–10 | **~$2.66/day average** |

---

## Live DB Findings (2026-04-12) — Root Cause Mechanism Confirmed

A live MySQL investigation via SSM port-forward identified the root cause mechanism. Full details in `2026-04-12-mysql-sms-root-cause.md`.

### tbl_alarm_alerts volume drop exactly matches SMS drop

`tbl_alarm_alerts` is the DB table that records alert events and drives the SMS notification pipeline. Its daily insert rate mirrors the CloudWatch SMS drop:

| Date | Total alert rows created |
|---|---|
| Mar 22–25 | 25–39/day (normal) |
| **Mar 26** | **14** (drop begins mid-day) |
| **Mar 27** | **4** |
| Mar 28–Apr 11 | 0–2/day |

All alert types dropped uniformly (DISCONNECT, BATTERY, ALERT, TAMPERED, RESET) — not a single-event-type bug. The **entire alert creation pipeline stopped** partway through March 26.

### CommunicationsQueue hypothesis — REFUTED for SMS

`tbl_communications_queue` has 899 rows, ALL type='email', ALL status=0 (undelivered). There are **no SMS rows**. This queue is an email queue. The `CommunicationsQueue` LaunchDarkly flag governs email batch delivery, not SMS. The SMS hypothesis based on this flag is incorrect.

### 4,808 device records mass-updated March 25–26

`tbl_alarms.updatedAt` shows 4,808 alarm sensor records updated in bulk on March 25–26 (normal: ~50–80/day). This is consistent with a database migration running as part of a code deployment. The same devices are still marked `connectedStatus=1` (connected), so this was not a mass-disconnect operation.

### Most likely proximate cause

A backend code deployment on March 26 broke the MQTT event → `tbl_alarm_alerts` insertion pipeline. Evidence:
- Drop began **mid-day March 26** (consistent with daytime deploy)
- Bulk `tbl_alarms` update consistent with migration script in a deploy
- `tbl_alarm_events_history` and `tbl_mqtt` both have 0 rows — consistent with those logging tables abandoned by a code refactor

> **Note:** Prior investigation found no CodeDeploy deployments to prod on March 26. However, ECS rolling updates (not CodeDeploy) would be the more common deployment mechanism. App team should check ECS task revision history.

### Additional finding: tbl_sms_allowed_list

A table `tbl_sms_allowed_list` exists with only 2 entries — both Indian test phone numbers from December 2022. If any SMS code path gates on this table, **no Australian customer would ever receive SMS**. This is a pre-existing concern to audit separately; it does not explain the March 26 drop (SMS was working normally before that date).

---



1. **Delivery rate is healthy throughout** — the messages that ARE sent get through (~94–100%). This means the Pinpoint/SNS infrastructure is functioning correctly.

2. **The problem is upstream** — fewer messages are being sent, not failing to send.

3. **No AWS-level configuration changes** were found:
   - No `sms-voice.amazonaws.com` CloudTrail events in March 23–28
   - No SNS write events (only `GetTopicAttributes`/`ListTagsForResource` reads)
   - Sender IDs (SENSORIQ, SENSOR, etc.) are all still configured and active
   - No opt-out configuration changes detected

4. **No prod CodeDeploy deployments** occurred in March 20–28 (all deployments in that window were to `Smoke-qa-API`, `qa-sso-api`).

5. **The drop is gradual, not instantaneous** — 585 → 452 → 352 → 342 (Mar 20–23) suggests it may have started slightly earlier than March 27.

---

## Possible Root Causes (to investigate)

| Hypothesis | How to verify |
|---|---|
| **Feature flag or app config change** silently disabled notifications for new installations/alerts | Check app config/feature flags deployed around March 20–27 |
| **`CommunicationsQueue` overnight batch** — queue logic or its inputs changed around March 26 | See note below; ask app team to trace what this queue sends and whether it changed |
| **Database migration or data change** removed/deactivated device records | Check `sensor-prod` DB for device/subscription record counts around that date |
| **Third-party integration** (e.g., push notification service, alarm monitoring service) changed | Review app integrations changelog |
| **SNS SMS monthly spend limit** hit and silently stopped sending | Check `SMSMonthToDateSpentUSD` — March total was ~$447, which is below any default limit |
| **User opt-out campaign** — mass opt-outs reduced the sendable audience | The opt-out list API returned an empty paginated result (may need paging); check app DB |
| **Alarm trigger logic change** — alarms are still registered but not triggering notifications | Check application logs for notification attempt counts vs. actual sends |

### CommunicationsQueue hypothesis (added 2026-04-12)

The Feature Flags doc (reviewed 2026-04-12) confirms a LaunchDarkly flag named `CommunicationsQueue` with description "Enables the overnight communications queue." This flag is **ON** in Dev, Sandbox, QA, and Production.

This overnight queue was not known during the initial SMS investigation. An overnight bulk-communication queue is a plausible pathway for the high-volume daily SMS sends seen before March 26 (350–585/day). If routine device status notifications, alerts, or tenant communications are processed in batch overnight rather than in real-time, then a change to the queue's logic, its data source, or the conditions that feed it could produce exactly the kind of gradual ramp-down and sudden sustained drop observed from March 20–27.

The flag has not been toggled off. The queue itself may be running but sending far fewer messages due to a logic or data change upstream of the queue.

---

## Recommended Next Steps (in priority order)

1. **Application team**: Check notification service code and config for changes deployed or pushed around March 20–27. The absence of prod CodeDeploy events does not rule out app-level config changes (environment variables, feature flags, Odoo config).

2. **Investigate `CommunicationsQueue` overnight batch:** Ask the application team to describe what the overnight communications queue sends (routine device status notifications, alert follow-ups, tenant SMS), whether that processing changed around March 26, and whether it is the primary path for the message types that dropped. The queue is confirmed ON in production via the Feature Flags doc.

2. **Database check** (`sensor-prod` MySQL): Count of active devices / registered alarm endpoints over time. If device count dropped, investigate why.

3. **Application logs** (on prod API instances or S3 log buckets): Search for lines like "SMS send", "notification", "alert" to count attempted sends vs skipped.

4. **Check CloudWatch Logs Insights** (once VPN is available):
   ```
   fields @timestamp, @message
   | filter @message like /SMS|notification|alert|send/
   | stats count() by bin(1d)
   | sort @timestamp asc
   ```

5. **Escalate to product owner**: Confirm whether the reduction is expected (e.g., a customer segment was deactivated, a large trial ended) or is a genuine bug.

---

## Impact Statement

This platform monitors residential smoke alarms. If SMS notifications are not being sent when alarms trigger, the core safety promise of the product is broken. This cannot be deferred. No cost optimization work should touch the SMS/notification infrastructure until this is resolved.

**Owner:** Engineering + Product  
**Urgency:** Immediate — prior to any other production changes

---

## Related AWS Resources

| Resource | Details |
|---|---|
| SMS namespace | `AWS/SMSVoice` |
| Prod sender ID | `SENSORIQ` (AU), `SENSOR` (AU, ID, others) |
| Other sender IDs | `SENSORIQSB`, `SENSORIQDEV`, `SENSOIQQA`, `SENSOR-SB`, `SENSOR-QA`, `SENSOR-DEV` |
| Opt-out list | `Default` (created 2022-06-13, no entries visible via API) |
| SNS SMS monthly limit check | March total: $447.67 — typical AWS default limit $1 (sandbox) or $100+ (approved). If in sandbox mode this would cap — but the March spend far exceeds $1, so sandbox is not the issue |
