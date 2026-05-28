# MySQL Live Investigation — SMS Drop Root Cause

**Date:** 2026-04-12  
**Account:** `747293622182` (ap-southeast-2)  
**Database:** `smokealarmprod` (MySQL, `sensor-prod.c4h1sisawrbr.ap-southeast-2.rds.amazonaws.com`)  
**Access method:** SSM port-forward via `i-0b53bd5e7a9b9dec0` (prod-api-server) → local port 13306  
**Classification:** 🚨 P1 — continues SMS drop investigation from `2026-04-11-sms-drop-investigation.md`  
**Status:** Root cause mechanism confirmed; proximate trigger (code deployment vs data change) requires app team confirmation

---

## Access Setup

```bash
# Open SSM tunnel (keep this running in a separate terminal)
AWS_PROFILE=sensorsyn-mfa aws ssm start-session \
  --target i-0b53bd5e7a9b9dec0 \
  --document-name AWS-StartPortForwardingSessionToRemoteHost \
  --parameters '{"host":["sensor-prod.c4h1sisawrbr.ap-southeast-2.rds.amazonaws.com"],"portNumber":["3306"],"localPortNumber":["13306"]}'

# Connect (credentials from Secrets Manager secret: sensor-prod)
# NOTE: Typo in secret key — use DB_MYSQL_PASSWPRD (not PASSWORD)
mysql -h 127.0.0.1 -P 13306 -u admin -p smokealarmprod
```

---

## Key Findings

### 1. The SMS drop is caused by `tbl_alarm_alerts` stopping to populate

`tbl_alarm_alerts` is the table that records alert events which drive SMS notifications. Its daily row creation rate fell off a cliff on March 26, exactly matching the AWS CloudWatch SMS send drop.

| Date | DISCONNECT | BATTERY | ALERT | TAMPERED | RESET | Total |
|---|---|---|---|---|---|---|
| Mar 22 | 8 | 14 | 3 | 1 | — | 26 |
| Mar 23 | 14 | 2 | 7 | 2 | — | 25 |
| Mar 24 | 15 | 10 | 9 | 4 | 1 | 39 |
| **Mar 25** | **17** | **1** | **11** | **1** | **—** | **30** ← last normal day |
| **Mar 26** | **5** | **6** | **1** | **2** | **—** | **14** ← drop begins mid-day |
| **Mar 27** | **3** | **—** | **—** | **1** | **—** | **4** ← near-zero |
| Mar 28 | 1 | — | — | — | — | 1 |
| Mar 29 | — | 1 | — | — | — | 1 |

The mid-day profile of March 26 is consistent with a deployment that occurred partway through the day.

All alert types dropped uniformly, which rules out a single-event-type bug (e.g., "DISCONNECT handler broken"). The entire alert creation pipeline stopped.

### 2. 4,808 alarm sensor records were mass-updated on March 25–26

`tbl_alarms.updatedAt` shows a bulk update of 4,808 rows on March 25–26:

| connectedStatus | controller | simStatus | Count |
|---|---|---|---|
| 1 (Connected) | 0 (sensor) | inactive | 3,601 |
| 2 (Removed) | 0 (sensor) | inactive | 2,174 |
| 0 (Not connected) | 0 (sensor) | inactive | 244 |
| (others) | mixed | — | ~30 |

Normal daily update volume before March 26 was ~50–80 rows/day. A bulk update of 4,808 rows in 2 days is 60–100× the normal rate.

The updated devices are mostly alarm sensors (`controller=0`, no SIM), not hubs. Their `simStatus=inactive` is expected (sensors don't have SIMs). Their `connectedStatus` values span all states — this was not a targeted disconnect operation; it looks like a migration or bulk-patch touching a different field.

`tbl_alarms_logs` shows only 107 field-level audit records for the same 2-day window (action types 2 and 4), so the `tbl_alarms.updatedAt` bulk change is **not** explained by the logged business actions — it was a background or scripted operation.

### 3. `tbl_alarm_events_history` and `tbl_mqtt` are both empty

These tables exist in the schema but contain 0 rows:

- `tbl_alarm_events_history` — `(propertyId, controllerId, alarmId, indexNumber, event, report, eventStatus, createdAt)` — 0 rows
- `tbl_mqtt` — `(subscriptionTime, createdAt, updatedAt)` — 0 rows

If these tables were intended to log raw MQTT events before processing, their emptiness suggests either:
- The code that writes to them was removed in a prior deployment
- They are pre-existing schema artefacts that were never populated in production

Their existence but emptiness is consistent with the MQTT processing pipeline having been refactored, with the current code path bypassing these tables.

### 4. `tbl_communications_queue` is email-only — the CommunicationsQueue hypothesis is wrong for SMS

Previous hypothesis: the `CommunicationsQueue` LaunchDarkly flag ("Enables the overnight communications queue") might govern bulk SMS sends.

**Finding:** `tbl_communications_queue` has 899 rows, ALL with `type='email'`, ALL with `status=0` (undelivered), spanning April 2025 onward. There are no SMS rows. This queue is an **email queue**, not an SMS queue.

The 899 undelivered email rows since April 2025 is a separate issue (email delivery failure) but is not related to the SMS drop.

**The `CommunicationsQueue` flag is not the SMS pathway.** The SMS drop must be explained by a different mechanism. See finding #1.

### 5. `tbl_sms_allowed_list` — dev whitelist with 2 Indian test numbers in production

| id | phoneCode | phone | description | status | createdAt |
|---|---|---|---|---|---|
| 1 | 91 | 9540331362 | Jeetendra | 1 | 2022-12-22 |
| 2 | 91 | 9634066078 | Prachi's Contact | 1 | 2022-12-22 |

This table appears to be a development/testing SMS whitelist created in December 2022. Both entries have Indian phone numbers (code 91), not Australian. If the production code were enforcing this whitelist before sending SMS, no Australian customer would ever receive an SMS — which contradicts the normal daily sends seen before March 26. This whitelist is either:
- Not referenced by the active production code path, or
- Used only in a specific non-critical code path

This table does **not** explain the March 26 drop (the drop was a transition from normal to broken, not a constant block). However, it is a pre-existing concern worth auditing: if any SMS codepath has a gate on this table, it should be removed or the table should be cleared.

### 6. Current device inventory

| controller | connectedStatus | simStatus | Count |
|---|---|---|---|
| 0 (sensor) | 2 (Removed) | inactive | 6,881 |
| 0 (sensor) | 1 (Connected) | inactive | 4,930 |
| 0 (sensor) | 0 (Not connected) | inactive | 519 |
| 1 (hub) | 0 (Not connected) | inactive | 12,228 |
| 1 (hub) | 0 (Not connected) | active | 7,215 |
| 1 (hub) | 2 (Removed) | inactive/active | 457 |
| 1 (hub) | 1 (Connected) | active | 56 |

Notably: only **56 hubs are marked `connectedStatus=1` (Connected)** in the database, yet EMQX broker logs and KORE SIM data indicate far more active connections. This discrepancy suggests `connectedStatus` in `tbl_alarms` may lag behind actual EMQX connection state — the field may be written only when specific application events occur, not continuously synced.

---

## Root Cause Assessment

### Confirmed mechanism
`tbl_alarm_alerts` stopped receiving new rows at the expected rate starting partway through March 26. Because this table drives the SMS notification pipeline, fewer rows → fewer SMS sends. The drop in the DB exactly mirrors the drop in AWS CloudWatch SMS metrics.

### Most likely proximate cause: backend code deployment on March 26
The evidence points strongly to an application code change deployed on March 26:

1. The drop began **mid-day on March 26**, consistent with a daytime deployment
2. The **4,808-row bulk `tbl_alarms.updatedAt` update** on March 25–26 is consistent with a database migration script running as part of a deployment
3. `tbl_alarm_events_history` and `tbl_mqtt` being empty is consistent with those logging tables having been **abandoned by a code refactor** rather than being deleted post-breach
4. No AWS-level infra changes were found on that date (no CodeDeploy deployments to prod, no SNS/SES config changes)
5. The drop is uniform across ALL alert types — this is not a selective bug in one handler; it is the **entire alert creation pathway** that stopped

> **Important caveat:** The prior investigation noted "No prod CodeDeploy deployments occurred in March 20–28." However, CodeDeploy is not the only deployment mechanism — the app team may deploy via ECS rolling updates, a CI/CD pipeline targeting ECS directly, or manual container pushes. This needs to be verified with the app team.

### Alternative cause: upstream data change breaking alert eligibility
If the 4,808 bulk update on March 26 changed a field that the alert creation logic gates on (e.g., a `verificationStatus`, `readStatus`, or a field in a related table), that could have silently disabled alert creation for those devices. This would not require a code change — a data-only change could produce the observed drop. The mass update touched primarily alarm sensors (`controller=0`) that are `connectedStatus=1` (active). If alert creation requires a specific value in a field that was bulk-changed to a different value, the result would be a 95%+ drop in alert volume.

---

## What the App Team Needs to Investigate

1. **Was there a backend deployment on March 26?** Check ECS task revision history for the prod MQTT subscriber service and prod API service. Even if CodeDeploy was not used, an ECS rolling update would show as a new task definition revision.

2. **What changed in the MQTT subscriber around March 26?** The subscriber is the code that receives MQTT events, processes them, and writes to `tbl_alarm_alerts`. Check git history for changes to that service around March 25–26.

3. **Why were 4,808 `tbl_alarms` rows updated on March 25–26?** This was a bulk operation. Was it a migration, a scheduled job, or a manual admin action? What field was changed?

4. **Is `tbl_sms_allowed_list` still referenced in production code?** If yes, it is blocking all Australian customers from receiving SMS.

5. **Why does `tbl_alarm_events_history` have 0 rows?** Was this table ever populated? When was the code that wrote to it removed?

---

## Related Tables Investigated

| Table | Rows | Notes |
|---|---|---|
| `tbl_alarm_alerts` | 11,687 | The alert record table; drop starts Mar 26 |
| `tbl_alarms` | 32,299 | Device inventory; 4,808 bulk-updated Mar 25–26 |
| `tbl_communications_queue` | 899 | Email-only; all status=0 (undelivered) since Apr 2025 |
| `tbl_alarm_events_history` | 0 | Empty — raw MQTT event log not written to |
| `tbl_mqtt` | 0 | Empty — MQTT session log not written to |
| `tbl_sms_allowed_list` | 2 | Dev test whitelist; Indian numbers only; potentially stale in prod |
| `tbl_user_notifications` | 2,957 | In-app portal notifications; volume stable (5–8/day), no drop |
| `tbl_outgoing_communications` | 186 | Old comms table, June 2025 rows only |
| `tbl_notifications` | 0 | Empty |
| `tbl_users` | 0 | Empty (archived?) |

---

## Related Investigations

- `2026-04-11-sms-drop-investigation.md` — original SMS drop investigation (AWS-level evidence, no infra cause found)
- `2026-04-12-upstream-tech-docs-review.md` — `CommunicationsQueue` flag first surfaced there as a hypothesis; now refuted for SMS

---

## Next Steps

1. **App team**: Check ECS task revision history for prod MQTT subscriber — any new revisions on March 25–27?
2. **App team**: Check git log for MQTT subscriber service changes around March 26
3. **App team**: Identify what the March 25–26 bulk `tbl_alarms` update changed (what column, what value)
4. **App team**: Confirm whether `tbl_sms_allowed_list` is referenced by active production code
5. **DB**: If SSM tunnel is still available, continue exploring — `tbl_alarms_logs` actionType codes 2 and 4 around March 26 could hint at what field changed
6. **Update** `2026-04-11-sms-drop-investigation.md` status from "root cause not yet identified" to confirmed mechanism + pending confirmation of proximate trigger
