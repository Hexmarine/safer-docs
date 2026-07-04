# Finding: Sensor discards all device-initiated LOW_BATTERY MQTT frames

**Date:** 2026-07-01
**Discovered via:** the forensic MQTT tap (`tbl_device_events`) — its first real catch
**Status:** confirmed (read-only); NOT yet changed — needs intent check before any fix
**Severity:** medium (battery-monitoring blind spot; NOT alarm blindness)

## What the tap caught

Hub `000464709926C0012406` publishes `{CMD:"LOW_BATTERY", BATTERY:0}` on
`sg/sas/resp/…` at **QoS 2**, repeatedly (01:07 / 01:13 / 01:15 / 01:52 UTC) — a dying
hub reporting a dead battery, and *only* emitting LOW_BATTERY (it no longer answers VERIFY
polls). Cross-checking Sensor:
- `tbl_alarm_logs`: **no** LOW_BATTERY row; the hub's last entry is a VERIFY at `00:19:06`.
- `tbl_alarms` (MySQL, id 5965): `batteryStatus:"100"`, `lowBattery:0`, `updatedAt 00:18:22`
  — Sensor believes the hub is at 100% and healthy.

A second hub (`002802531840…`, child index 1) shows the same pattern
(`LOW_BATTERY BATTERY:10–15` on the wire; Sensor DB stale/healthy).

## Root cause (code, not QoS)

`code/sensor-alarm-backend/src/services/mqtt/subscriber.ts:197-199`, at the very top of the
inbound MQTT message handler:

```js
if (message.CMD == "LOW_BATTERY") {
  return;   // discards the frame before any logging / column update / email
}
```

There is **no `LOW_BATTERY` case in the main dispatch chain** (lines 200–671). The
LOW_BATTERY logging seen at ~`subscriber.ts:4023` (`HUB_LOW_BATTERY`, `lowBattery:"1"`) is
reached via a different path, **not** device-initiated frames — those hit the early return.

So Sensor learns of low battery **only** from its BATTERY-poll path (`updateAlertOnHub`,
`batteryStatus <= 15`). A hub that dies without/between poll responses is invisible.

## This is NOT the QoS-0 drop

Distinct from `2026-06-30-sensor-mqtt-qos0-finding.md`: the frame is *received then
discarded by code*, not lost in transit. The 30-min "LOW_BATTERY tap 2 / Sensor 0"
discrepancy is fully explained by this `return`, so it does **not** evidence a QoS-0 drop
(that remains unmeasured — needs a longer window / a disruption).

## How `batteryStatus` / `lowBattery` ARE maintained (pull-only)

`batteryStatus` (the raw %) is written **only from device replies to polls** — there is no
other path:
- **VERIFY reply** (`CMD:VERIFY, STATUS:1`) → `verifyAlarm({batteryStatus: message.BATTERY, …})`
  (`subscriber.ts:204-212`) — the dominant path, riding the hub-heartbeat / VERIFY
  connectivity sweep.
- **ADD reply** (`subscriber.ts:308`) and **explicit BATTERY-poll reply** (`:550/560/575`).

`lowBattery` (the derived flag) is set: directly to `"1"` on a `BATTERY <= 15` frame
(`subscriber.ts:551/555`); or recomputed by `updateAlertOnHub` (`alarms.entity.ts:2608`,
called from the VERIFY/ADD/TEST/ALERT/TAMPERED handlers), which rolls up the hub's children
via `CAST(batteryStatus AS UNSIGNED) <= 15` (line 2650 — this *is* the remediation for the
string-compare systemic bug, so the rollup computes correctly).

**Consequence — the structural blind spot.** Battery monitoring is entirely **pull-based**
(poll → device replies → store). The single **push** channel — a device spontaneously
sending `LOW_BATTERY` when it is too degraded to keep answering polls — is the one discarded
at `subscriber.ts:197`. So a hub that dies (`BATTERY:0`, no longer answering VERIFY) has all
three pull write-paths unavailable *and* its push fallback disabled; `batteryStatus` freezes
at its last good poll value and reads healthy indefinitely. The rollup is correct — it is
computing on **stale input** the pull-only model cannot refresh. The gap is worst exactly
when it matters most: a dying hub.

## Before any fix

- The `return` is **likely a deliberate band-aid** for the systemic spurious-`lowBattery`
  bug (~2,340 hubs falsely flagged; see MEMORY `hub-lowbattery-string-compare-systemic-bug`).
  Removing it could resurrect a false-alert storm. **Check git history / intent first.** The
  real fix is coupled to the `lowBattery` string-compare (`CAST … UNSIGNED`) remediation.
- Scope is bounded: LOW_BATTERY only. Fire/smoke alarms use `CMD:ALERT` (handled at
  `subscriber.ts:407+`) and are **not** discarded.
- Class-C (existing Sensor behaviour) — any change needs the regression guard + approval.
