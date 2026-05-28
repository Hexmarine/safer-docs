# Alarm Event Traces

- Date: 2026-04-09
- Scope: trace what happens in the backend after a hub-reported alarm event is received over MQTT
- Related systems: Sensor Hub or controller, child alarms, MQTT subscriber, alarm state tables, Mongo alarm logs, notifications, socket updates

## Objective

Show the concrete backend flow after a device event arrives, with emphasis on how the system resolves the hub, resolves the child alarm, updates state, logs the event, and decides whether to notify people.

## Sources Used

- Repository files:
  - `code/sensor-alarm-backend/src/services/mqtt/subscriber.ts`
  - `code/sensor-alarm-backend/src/entities/alarms.entity.ts`
  - `code/mqtt-prototype/firmware.md`
  - `docs/investigations/2026-04-09-platform-architecture-and-data-flow.md`
  - `docs/investigations/2026-04-09-mqtt-device-communications.md`
- AWS commands or consoles:
  - none needed for this pass
- External references:
  - `docs/upstream/Technical Documents/SENSORGLOB-Firmware & Protocol Specification` (reviewed 2026-04-12)
  - `docs/upstream/Technical Documents/SENSORGLOB-Event types` (reviewed 2026-04-12)
  - See cross-reference: `docs/investigations/2026-04-12-upstream-tech-docs-review.md`

## Findings

- The hub is the cloud-facing sender for all MQTT events. Child alarms are not first-class cloud clients.
- The backend dispatch pipeline is:
  1. resolve hub by topic serial
  2. resolve child alarm by `INDEX` when applicable
  3. update current state in MySQL
  4. update alert-summary state and realtime socket counts
  5. run notification policy and flapping suppression
  6. append an alarm event log in Mongo
- The same general pattern is reused for alerting, tamper, disconnect, reconnect, battery, and test results.
- The most important entity boundary is:
  - hub identity from MQTT topic
  - child alarm identity from controller-local `INDEX`
- `ALARMS` inventory sync is what keeps that `INDEX -> alarm serial` mapping current.

### Shared dispatch model

- The MQTT subscriber receives hub responses on a shared subscription and extracts the hub serial number from the topic path.
- Hub resolution:
  - `getControllerBySerialNumber` looks up the active hub record where `controller = 1`
- Child alarm resolution:
  - `getAlarmByIndexNumber` looks up the child alarm where `controller = 0`, `controllerId = <hub id>`, and `indexNumber = <INDEX>`
- Child alarm inventory is maintained by `syncAlarmList`, which parses `ALARMLIST` values like `0;AAAA,1;BBBB`.
- State writes are split between:
  - `updateAlarmUsingIndex` for the child alarm
  - `updateAlertOnHub` for the hub-level aggregate state
- Both update helpers also emit socket refresh signals and maintain `AlarmAlerts` rows for the alert center.
- Notifications are not sent blindly:
  - `checkAlarmAndScheduleNotification` resolves hub, child alarm, and job status first
  - `sendNotification` then loads property details, alert configuration, and flapping rules before deciding who gets email, SMS, or push
- Event history is persisted through `addEventLog`, which stores property, agency, hub, alarm, status, battery, and event details in the alarm log collection.

### Event source and purpose

| Event | Sent by | Resolves to | Main purpose |
| --- | --- | --- | --- |
| `VERIFY` | hub | hub | confirm hub is online and report hub battery/power |
| `ALARMS` | hub | hub plus all child alarms | sync the current alarm inventory under that hub |
| `ADD` | hub after server command | child alarm | confirm a new alarm was linked and assign its hub-local `INDEX` |
| `REMOVE` | hub after server command | child alarm | confirm an alarm was unlinked |
| `TEST` | child alarm via hub or server-triggered via hub | child alarm | compliance and health testing result |
| `ALERT` | child alarm via hub | child alarm plus hub aggregate | actual safety incident state on or off |
| `TAMPERED` | child alarm via hub | child alarm plus hub aggregate | enclosure or mounting tamper state |
| `DISCONNECT` | child alarm via hub | child alarm plus hub aggregate | heartbeat failure or offline child alarm |
| `RECONNECT` | child alarm via hub | child alarm plus hub aggregate | heartbeat restored |
| `BATTERY` | hub or child alarm via hub | hub or child alarm | current battery percentage |
| `POWER` | hub | hub | mains versus backup battery |
| `HUSH` | child alarm via hub | child alarm plus hub aggregate | hush button state change |
| `RESET` | hub | hub and logs | factory reset / major lifecycle event |

## Concrete Traces

### Trace 1: `ALERT` state turns on

Example payload shape from the protocol:

`{"CMD":"ALERT","INDEX":2,"ALARMTYPE":1,"STATE":1}`

Step-by-step:

1. The backend receives the MQTT message on the hub response topic.
2. It extracts `controllerSerialNumber` from the topic.
3. In the `ALERT` branch, it prepares:
  - `updateAlarmUsingIndexData` for child alarm `INDEX = 2`
  - `updateAlertOnHubData` for the hub aggregate
4. It calls `checkAlarmAndScheduleNotification(payload, "ALERT")`.
5. That helper resolves:
  - the hub by serial number
  - the child alarm by `controllerId + INDEX`
  - the install or job context for that alarm
6. If the job is completed, it updates child alarm state and hub aggregate state:
  - child `alertStatus`
  - hub aggregate `alertStatus`
7. `updateAlarmUsingIndex` also:
  - updates or creates `AlarmAlerts` state for the child alarm
  - emits socket refresh events for sidebar, alert count, and dashboard count
  - if the event is `ALERT`, it forces the child alarm back to `CONNECTED` and clears disconnect alerts for that alarm and its hub
8. `sendNotification` then:
  - resolves the property
  - loads alert configuration
  - checks flapping suppression
  - determines message templates and recipients
  - emits email, SMS, or push if allowed
9. `addEventLog` stores a Mongo alarm log including:
  - property
  - agency
  - hub
  - child alarm
  - event type `ALERT`
  - `ALARMTYPE`
  - state and battery details when present

Operational purpose:

- Mark the exact child alarm as sounding or cleared
- Reflect the incident in the UI immediately
- Trigger customer or agency notification policy
- Preserve a durable audit trail

### Trace 2: `TAMPERED` turns on

Example payload shape from the protocol:

`{"CMD":"TAMPERED","INDEX":2,"STATE":1}`

Step-by-step:

1. The backend resolves the hub by topic serial and the child alarm by `INDEX`.
2. The `TAMPERED` branch prepares child and hub update payloads with `tamperedStatus`.
3. It calls `checkAlarmAndScheduleNotification(payload, "TAMPERED")`.
4. That helper again resolves:
  - hub
  - child alarm
  - job status
5. For most alarms on completed jobs, tamper notifications are immediate.
6. For water leak detectors (`A004`), tamper-on is delayed by 10 minutes before notifying.
7. State updates propagate through:
  - child alarm row
  - hub aggregate row
  - `AlarmAlerts`
  - socket updates
8. `sendNotification` applies property-specific tamper messaging rules.
9. `addEventLog` stores the tamper event in Mongo alarm logs.

Operational purpose:

- Distinguish genuine safety events from maintenance or physical interference events
- Alert the relevant parties when a device is no longer physically secure
- Preserve compliance evidence that tamper conditions occurred and were handled

### Trace 3: child alarm battery falls to 15% or below

Observed backend path:

- `LOW_BATTERY` as a standalone command is returned early by the MQTT subscriber and not processed further in the main branch.
- In practice, the only low-battery path that is handled is `BATTERY` with `BATTERY <= 15`.

**⚠ Confirmed implementation gap (2026-04-12):** The firmware spec and Event Types doc define `LOW_BATTERY` as a real standalone command with multiple thresholds:
- Hub `LOW_BATTERY` is emitted at **15%, 10%, and 5%** battery (three separate re-alerts).
- Alarm `LOW_BATTERY` is emitted at **5%** only.

The backend's early return on `LOW_BATTERY` means that only the first hub threshold (15%) is ever reported — via the `BATTERY <= 15` path — but the 10% and 5% crossings, which the firmware spec emits as standalone `LOW_BATTERY` events, are silently discarded. Operators and tenants would not receive the second or third low-battery warning for hub batteries, even though the firmware spec requires them.

Example payload shape:

`{"CMD":"BATTERY","INDEX":2,"BATTERY":15}`

Step-by-step:

1. The backend resolves hub and child alarm identity.
2. In the `BATTERY <= 15` branch:
  - the child alarm battery is updated
  - `lowBattery` is marked on the child alarm path
  - the hub aggregate is updated
3. The payload is enriched with:
  - `Index`
  - `battery`
  - raw message details
4. `sendNotification` is called directly.
5. Inside `sendNotification`, the case is normalized to `low_battery` so property alert configuration can apply the right rules.
6. `updateAlarmUsingIndex` treats low battery as a low-battery alert type and updates socket counts.
7. `addEventLog` writes the battery event and sets a future `checkDate` seven days ahead for low-battery follow-up.

Operational purpose:

- Turn raw percentage readings into a support and maintenance signal
- Keep the UI and alert center accurate
- Trigger battery replacement or service workflows before the alarm becomes unavailable

### Trace 4: why `ALARMS` matters before child events

`ALERT`, `TAMPERED`, `DISCONNECT`, `RECONNECT`, `TEST`, and child `BATTERY` depend on the `INDEX -> alarm` mapping being correct.

That mapping is maintained when:

1. The server sends `VERIFY`
2. The hub responds to `VERIFY`
3. The server requests `ALARMS`
4. The hub returns the current inventory list
5. `syncAlarmList` parses each `<index>;<serial>` pair
6. `checkAlarmAndUpdate` either:
  - updates an existing alarm with that index
  - or creates a new child alarm record under the hub
7. Any child alarms no longer present are marked deleted or removed

Operational purpose:

- Keep child alarm identity stable enough that later `INDEX`-based events can be routed to the correct database row
- Recover from pairing changes, reset events, or inventory drift at the hub

**Alarm index re-use (confirmed 2026-04-12 from firmware spec):** when a linked alarm is removed, its index is immediately freed. New alarms fill the lowest available index. Example: remove index 1 from a hub with alarms at indices 0, 1, 2 → add two new alarms → they take indices 1 and 3. This means index-to-serial mapping is not stable across pairing changes. The `VERIFY→ALARMS→BATTERY` sync must be completed after any removal or re-pairing to prevent routing future `INDEX`-keyed events to a stale serial.

## Risks Or Constraints

- The traces above are code-derived and represent intended behavior, not a captured production message trace.
- **`LOW_BATTERY` confirmed gap (2026-04-12):** the firmware spec defines `LOW_BATTERY` as a real standalone event with hub thresholds at 15%, 10%, and 5%, and alarm threshold at 5%. The backend returns immediately on standalone `LOW_BATTERY`, so only the 15% hub crossing (via the `BATTERY <= 15` path) is handled. The 10% and 5% re-alerts are silently discarded.
- **`FAULT` confirmed gap (2026-04-12):** the firmware spec defines `FAULT CODE 10` as the event emitted when a smoke alarm's sensing chamber is obstructed and can no longer adequately detect smoke (cleaning required). There is no handler for `FAULT` in the main MQTT subscriber. If this event fires, it is silently discarded — no notification, no maintenance job, no audit log.
- **`HUSH`/`ALERT` STATE encoding (2026-04-12):** the firmware spec uses `BoolState 0=Undefined, 1=On, 2=Off`. The "off" signal is `STATE:2`, not `STATE:0`. Backend branches that clear alert or hush state must handle `STATE:2` correctly. This has not been verified from code.
- Some helper behavior, such as template generation and flapping suppression detail, is broad and business-heavy, so this note focuses on the main dispatch path rather than every downstream branch.

## Cost Optimization Opportunities

- None directly from the trace itself, though the notification and logging richness suggests any logging-cost reduction must be balanced against compliance and audit requirements.

## Recommended Next Steps

- Verify backend handling of `STATE:2` (Off) in the `ALERT` and `HUSH` branches — confirm that clearing an alert sends `STATE:2` and that the backend does not misread it as `STATE:0`.
- Trace `FAULT` and `REBOOT` end to end to confirm whether they are partially implemented, handled elsewhere, or effectively dormant. Now that `FAULT CODE 10` (cleaning required) is confirmed as a real sensor-emitted event, the dormant handler represents a production gap.
- Trace one full notification path, including the concrete email, SMS, and push template resolution, for an `ALERT` event.
- Verify whether the backend adds a `LOW_BATTERY` handler for the 10% and 5% hub thresholds, or whether the standalone `LOW_BATTERY` path was intentionally replaced by the `BATTERY <= 15` path only.

## Open Questions

- Does the backend correctly handle `STATE:2` as the "clear" (Off) signal for `ALERT` and `HUSH`, or does any branch treat `STATE:0` or `!STATE:1` as the off-path?
- Is the standalone `LOW_BATTERY` handler gap intentional (i.e. the team decided `BATTERY <= 15` is enough), or is it an oversight? If intentional, the firmware spec and Event Types doc should be updated.
- Are `FAULT` and `REBOOT` responses used in production but processed in another code path, or has implementation drifted from the written protocol?
- How often does the `INDEX -> serial` mapping drift in practice, and is `VERIFY -> ALARMS -> BATTERY` reliably completed after hub resets and pairing churn?
