# Alert types map & proposal to split grouped alerts (2026-07-03)

Task: identify every alert type and its trigger, then propose splitting the
grouped ones so an agency can decide who gets the notification for each
individual alert type.

All references are `code/sensor-alarm-backend` unless noted. Verified against
current working tree.

## 1. How routing works today

Per-property JSON column `tbl_properties.alertConfiguration` (free-form —
`Joi.object()` at `routes/users/v1/properties.routes.ts:877`), keyed by
**notification type** → **audience role** → **channel**:

```json
{ "<type>": { "agent":   [{ "email": b, "sms": b, "push": b }],
              "agency":  [...], "tenants": [...], "landlord": [...] } }
```

- Only element `[0]` of each role array is ever read
  (`services/mqtt/subscriber.ts:1025,1130`, etc.).
- Recipients are **roles, not people**: the property's assigned agent, the
  agency master email, all active tenants, the landlord/asset owner
  (+ `contractor` for the job `communications_*` keys). There is no
  per-person targeting today.
- Defaults: `DEFAULT_PROPERTY_CONFIGURTION` (`src/constants/app.ts:1300`);
  agencies can override their own default via
  `PUT /users/settings/update-default-configuration`; per-property override via
  `PUT /users/properties/update-configuration`.
- Portal UI (`sensor-angular` … `property-details/view/configuration/`) is
  **fully data-driven**: it iterates `configuration | keyvalue`
  (`configuration.component.html:48`) and renders row labels from the key with
  `_` → space. New keys appear automatically.

## 2. Inventory of alert/notification types

### A. Event-driven (hub MQTT → `sendNotification`, `alertConfiguration`-gated)

Routing key = `payloadData.type.toLowerCase()` (`subscriber.ts:993-1012`).

| Config key | Trigger | Notes |
|---|---|---|
| `alert` | MQTT `ALERT` event from hub | **GROUPED — covers every physical alarm type** (see §3) |
| `tampered` | MQTT `TAMPERED` first occurrence; also water-leak sensor tamper ≥10 min (`subscriber.ts:835-844`) | all device models share it |
| `tampered_for_15_minutes` | repeat `TAMPERED` while a prior tamper log exists (`subscriber.ts:845-848`) | key is back-filled into old configs at `controllers/alarms.controller.ts:2061` — **precedent for injecting a new key** |
| `disconnect` | hub heartbeat/LWT flip to disconnected | off by default; one-shot (`disconnectionEmailSent`) |
| `reconnect` | hub comes back | agent email on by default |
| `low_battery` | `BATTERY` event ≤15%, or `VERIFY` reporting ≤5% (`subscriber.ts:994-1003`) | hub vs alarm both funnel here; >5% suppressed for agent/agency |

### B. Cron re-notifiers (external cron → `GET /admins/mqtt/start-cron` → `cronFunction`, `subscriber.ts:2910`; still `alertConfiguration`-gated)

| Config key | Cadence |
|---|---|
| `tampered_for_2_days` | still tampered after 2 days (key commented out of defaults; `tampered_for_two_days` alias handled at `subscriber.ts:1009-1011`) |
| `disconnect_every_day` | daily while still disconnected (agent email on by default) — currently paused via `#SWAP-PAUSE` |
| `low_battery_for_week` | still low after 7 days |

### C. Request/lifecycle communications (config-gated, `communications_*` keys)

Property invite (+1-week reminder), invite accepted on behalf of AO, alarm test
complete, EN scheduled/rejected by agency, scheduled test notification,
installation job confirmation, invite contractor, job assigned/updated/closed.
Single-role each; already individually configurable — no split needed.

### D. NOT gated by `alertConfiguration` at all (cron/API-driven)

S162 "Scheduled Test unsuccessful" + upcoming-test reminders
(`test-alarm-cron`/`test-alarm-notify-cron` — only gates are `status=ACTIVE` +
`alarmTestDate`), lease-expiry (S149), overdue-job reminders, run sheets,
summary emails, account/password emails. If the agency expects the new
per-alert-type screen to control *these*, that is a separate (larger) piece of
work — call it out explicitly in any customer conversation.

## 3. The grouping problem

The single `alert` key routes **all** physical alarm activations identically.
The firmware payload carries `ALARMTYPE` (`ALERT_TYPE`, `constants/app.ts:1701`:
smoke=1, CO=2, water-leak=3, gas=4, vibration=5, door=6, CO2=7, refrigerant=8)
and the device model is derivable from the serial prefix
(`A001` smoke, `A002` CO, `A003` smoke+CO combo, `A004` water leak).

Today the subtype only changes the message *wording*
(`subscriber.ts:4208-4219`), never the recipients. So an agency that wants
"smoke → tenants SMS + agency email, but water-leak → agent only" cannot
express it.

Bug found while mapping (worth fixing in the same change): the wording is
derived from the device model, not `ALARMTYPE`, so a **smoke** activation on an
A003 combo alarm is described as "carbon monoxide" (`subscriber.ts:4215-4216`).

Lesser groupings (defer unless asked): `low_battery` and `tampered` mix hub and
alarm devices; `disconnect` is hub-only so nothing to split.

## 4. Proposal: split `alert` by alarm type (Class-B additive)

New config keys, derived at routing time from `ALARMTYPE` (fallback: serial
prefix):

- `alert_smoke` (ALARMTYPE 1; A001, A003-smoke)
- `alert_carbon_monoxide` (ALARMTYPE 2; A002, A003-CO)
- `alert_water_leak` (ALARMTYPE 3; A004)
- (future types 4-8 fall through to legacy `alert` until we add keys)

**Backwards compatibility — no data migration required.** In
`sendNotification` (`subscriber.ts:~993`), compute
`caseType = "alert_" + subtype` and fall back:

```
notificationConfiguration = configData[splitKey] || configData["alert"]
```

18k+ properties keep their stored JSON and behave exactly as today until an
agency touches the new toggles. This mirrors the existing
`tampered_for_15_minutes` back-fill precedent (`alarms.controller.ts:2061`).

Touchpoints:

1. `constants/app.ts` — add the three keys to `DEFAULT_PROPERTY_CONFIGURTION`
   (+ `_CUSTOM`) and `ENotificationType`. Default matrix: see §5.
2. `services/mqtt/subscriber.ts` — caseType derivation + fallback (above);
   also fix the A003 wording to use `ALARMTYPE`.
3. API — none: both update endpoints accept free-form objects.
4. Portal UI — none for basic rendering (keyvalue-driven); optionally keep the
   legacy `alert` row hidden once split keys exist, and provide per-key
   `previewData`.
5. Regression class: **B (additive touch)** — run sensor-regression-guard
   before commit; existing `alert`-only configs must route byte-identically.

## 5. Proposed defaults for the split keys

<!-- TODO(human): fill the default role×channel matrix for the three new keys -->

## 6. Out of scope / follow-ups

- Per-person (not per-role) recipients — needs a schema change, not just keys.
- Splitting `tampered`/`low_battery` by device class.
- Bringing the non-config-gated comms (§D) under `alertConfiguration`.
