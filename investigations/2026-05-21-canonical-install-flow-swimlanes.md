# Canonical install flow — swim-lane timelines

Date: 2026-05-21

Side-by-side flows for the **canonical / original** smoke-alarm installation
flow (the on-site native-app flow). NOT the new safer-ops pre-paired-kit flow.

See also:
- `docs/diagrams/flow-device-installation.puml`
- `docs/diagrams/flow-job-lifecycle.puml`
- `docs/diagrams/flow-alarm-test.puml`
- `docs/investigations/2026-04-09-platform-architecture-and-data-flow.md`
- `docs/investigations/2026-04-09-alarm-event-traces.md`
- `docs/investigations/2026-04-09-mqtt-device-communications.md`

Note on SIM: production fleet uses **Omondo pre-activated SIMs**. There is no
SIM activation step on the install critical path. The KORE flow described in
`2026-05-02-sim-activation-flow.md` is historical.

Note on UI updates: **the native installer app does NOT use sockets.** Real-time
sockets exist only in the `sensor-angular` admin portal. The native app sees
state changes via (a) HTTP response of the call it made, (b) FCM/APNs push
notifications from backend, and (c) manual refresh / re-polling. Where this
doc says "appears via socket push" on an Admin/Agency row, that's the portal.
Installer rows that previously implied sockets actually mean "the next API
response or a push notification reflects the new state".

---

## 1. Installer ↔ Device (2-lane)

Installer (mobile app over HTTPS) on the left, Hub (LTE → MQTT broker) on the
right. Backend mediates implicitly. Arrows show direction from the
column's perspective (`→` outbound, `←` inbound).

| # | Installer (native app, HTTPS) | Hub (LTE → MQTT broker) |
|---|---|---|
| 0 | Receives assigned job: `GET /users/v1/jobs/app` | Powered off |
| 1 | Arrives on-site, mounts hub, plugs in power | Boots, attaches LTE (Omondo SIM, pre-active), opens MQTT session, subscribes to `sg/sas/cmd/<serial>` |
| 2 | Taps "Add hub" → `POST /users/v1/alarms` with `controllerData` | — |
| 3 | (waiting) | ← `{"CMD":"VERIFY"}` on `sg/sas/cmd/<serial>` |
| 4 | (waiting) | → `{"CMD":"VERIFY","STATUS":1,"BATTERY":..,"POWER":..}` on `sg/sas/resp/<serial>` |
| 5 | Hub appears as **CONNECTED** in app (HTTP response of step 2 returns once backend confirms; subsequent state changes via push notifications + manual refresh) | Idle, listening |
| 6 | Taps "Add sensor" for each location → `POST /users/v1/alarms` with `alarmsData[]` | — |
| 7 | (waiting, 10 s per sensor) | ← `{"CMD":"ADD","ALARMSERIAL":"AAA"}` (one per sensor) |
| 8 | (waiting) | RF-pairs sensor at 433 MHz, assigns local INDEX |
| 9 | (waiting) | → `{"CMD":"ADD","ALARMSERIAL":"AAA","STATUS":1}` |
| 10 | Sensor row flips to CONNECTED. If no reply in 10 s → push: "device not paired" | (silent on failure — no auto-retry observed) |
| 11 | All sensors added; UI shows full list | Idle |
| 12 | (backend kicks inventory sync — installer doesn't trigger explicitly) | ← `{"CMD":"VERIFY"}` then ← `{"CMD":"ALARMS"}` |
| 13 | (waiting) | → `{"CMD":"ALARMS","ALARMLIST":"0;AAA,1;BBB,..."}` |
| 14 | App shows INDEX-mapped sensor list (durable IDs now set) | ← `{"CMD":"BATTERY"}` |
| 15 | Battery % populates per sensor | → `{"CMD":"BATTERY","INDEX":n,"BATTERY":..}` per sensor |
| 16 | Taps **Test** on a sensor → `POST /users/v1/alarms/test-alarm` | ← `{"CMD":"TEST","INDEX":n}` |
| 17 | (waiting) | RF-fires sensor; sensor beeps/simulates smoke |
| 18 | Test result toast: pass / fail | → `{"CMD":"TEST","INDEX":n,"STATUS":1\|2\|0}` |
| 19 | Adds notes + photos, taps **Complete** → `PUT /users/v1/jobs/complete` | Idle, remains online |
| 20 | Job → COMPLETED; alarm goes **live** for the property | Continues normal operation: heartbeat, alerts, scheduled tests |

### Reading notes

- **Backend orchestration is async**: each HTTP call publishes MQTT commands
  and returns; the matching MQTT response arrives independently and is
  correlated via the Redis `<hub-serial>_HUB_VERIFY` flag
  (`alarm.controller.ts:121, 138, 154`). "Waiting" rows are real wall-clock
  waits but the HTTP call itself has already returned.
- **Slow / flaky step** is steps 7–10: RF pairing happens on the hub, off-band
  from MQTT. This is the on-site bottleneck the new safer-ops flow is trying
  to remove.
- **Index assignment** is hub-local (step 8) but only becomes known to
  backend at step 13 (`syncAlarmList`). Every later op (TEST, ALERT, BATTERY)
  uses INDEX, not serial.

---

## 2. Admin/Agency ↔ Installer ↔ Device (3-lane)

Admin / Agency (sensor-angular web portal) covers Agency + TradePerson office
actions; Installer is the on-site ServiceStaff in the native app; Device is
the hub + RF-paired sensors over MQTT.

| # | Admin / Agency (web portal, HTTPS) | Installer (native app, HTTPS) | Device (hub + sensors, MQTT) |
|---|---|---|---|
| 0 | Agency creates job: `POST /users/v1/jobs` (type=INSTALLATION, property, date, tradeperson). Status = **PENDING** | — | Sits in warehouse, unpowered |
| 1 | (job assigned to TradePerson) | — | — |
| 2 | TradePerson accepts: `PATCH /jobs/status`. Status → **ACCEPTED** | — | — |
| 3 | TradePerson assigns ServiceStaff to job | — | — |
| 4 | (optional) Agency schedules Notice of Entry → **NOE_SCHEDULED** | — | — |
| 5 | — | Sees job in `GET /users/v1/jobs/app`; travels to property | — |
| 6 | — | Starts work in-app: `PATCH /users/v1/jobs/update-start-time` → status → **IN_PROGRESS** | — |
| 7 | — | Mounts hub, powers it up | Boots, LTE attaches (Omondo, pre-active), MQTT subscribes to `sg/sas/cmd/<serial>` |
| 8 | — | "Add hub": `POST /users/v1/alarms` (controllerData) | ← `{"CMD":"VERIFY"}` |
| 9 | (sees hub appear under property via **socket** — admin portal has socket.io against `socketUrl`) | Sees hub flip to CONNECTED (HTTP response + push, no socket) | → `{"CMD":"VERIFY","STATUS":1,"BATTERY":..,"POWER":..}` |
| 10 | — | "Add sensors": `POST /users/v1/alarms` (alarmsData[]) | ← `{"CMD":"ADD","ALARMSERIAL":...}` × N |
| 11 | — | Waits (10 s/sensor); failures → push "device not paired" | Hub RF-pairs each sensor, assigns local INDEX |
| 12 | — | Sensor rows flip to CONNECTED | → `{"CMD":"ADD",..,"STATUS":1}` per sensor |
| 13 | — | (backend kicks inventory sync) | ← `{"CMD":"VERIFY"}` then ← `{"CMD":"ALARMS"}` |
| 14 | (live sensor list visible in portal) | App shows INDEX-mapped list | → `{"CMD":"ALARMS","ALARMLIST":"0;AAA,1;BBB,..."}` |
| 15 | — | Battery % populates | ← `{"CMD":"BATTERY"}` → per-sensor responses |
| 16 | — | Taps **Test** on each sensor: `POST /users/v1/alarms/test-alarm` | ← `{"CMD":"TEST","INDEX":n}` → sensor beeps → `STATUS:1\|2\|0` |
| 17 | — | Captures notes + photos, taps **Complete**: `PUT /users/v1/jobs/complete` | Idle, remains online |
| 18 | Sees job → **COMPLETED**; alarms go live on property; agency dashboards update | Job closed in app | Continues normal operation: heartbeats, alerts, scheduled compliance tests |
| 19 | (later) Agency reviews compliance, runs reports, raises maintenance/replacement jobs as needed | — | Emits ALERT / FAULT / LOW_BATTERY events as they occur (see gaps: FAULT 10 and 10/5 % LOW_BATTERY are dropped by backend) |
| 20 | (cancel path, any time pre-complete) `PATCH /jobs/status` → **CANCELLED** with reason | If unstarted, job drops off list | n/a |

### Reading notes

- **Admin/Agency activity is bursty**: heavy at steps 0–4 (job setup) and
  step 18+ (oversight/compliance/reporting), almost silent during the on-site
  window. They're observers via sockets during 7–17, not active participants.
- **Installer column** is the on-site critical path; it's also where every
  "slow/flaky" minute is spent (RF pairing in step 11). This is what the new
  safer-ops flow moves to the depot.
- **Device column is fully reactive** in the canonical flow — it only acts in
  response to commands; the hub never initiates anything install-related
  itself.
- **TradePerson** isn't a separate column because in code they're a role on
  the Agency side of the portal, sharing the same web app and endpoints. Pull
  them out if you want a 4-lane version.

---

## 3. Post-install (already-connected device) — operational flow

Inferred from `alarm-event-traces.md`, `flow-alarm-event.puml`,
`mqtt-device-communications.md`, and `subscriber.ts`. Covers the steady-state
once a property is live: alerts, tests, battery reports, disconnect/reconnect,
and how Admin/Agency interact with a device that is already paired (no
installer present).

In this phase the **Device is the initiator** for safety-critical events; the
**Admin/Agency portal** is the primary human surface; the **Installer column
is mostly empty** (only reappears for maintenance / replacement / removal
jobs, which re-enter the install flow above).

### 3a. Real-time alert (smoke / heat / CO)

| # | Admin / Agency (web portal + push) | Backend | Device (hub + sensors, MQTT) |
|---|---|---|---|
| 0 | — | Idle, MQTT subscribed to `$share/sensorGroup/sg/sas/resp/+` | Sensor detects smoke; RF-signals hub |
| 1 | — | — | Hub → `{"CMD":"ALERT","INDEX":n,"STATE":1,"TYPE":..}` on `sg/sas/resp/<serial>` |
| 2 | — | `subscriber.ts` parses, resolves hub by serial, looks up sensor by INDEX, writes `AlarmAlerts` (active alert), appends `AlarmLogs`, **emits socket event** (consumed by `sensor-angular` admin portal — `SocketService` confirmed in `sensor-angular/src/app/modules/shared/services/socket.service.ts`) | — |
| 3 | Live alert appears on agency dashboard (socket); push/SMS to landlord/tenant fires (and FCM/APNs push to installer app if assigned) | — | — |
| 4 | Agency can acknowledge / annotate in portal | — | — |
| 5 | — | — | When event clears: hub → `{"CMD":"ALERT","INDEX":n,"STATE":2}` (per firmware spec — backend handling of `STATE:2` not verified, see gaps) |
| 6 | Alert clears in UI (when STATE handled correctly) | Updates `AlarmAlerts.eventStatus`, logs clear | Idle |

### 3b. On-demand compliance test (initiated by agency)

| # | Admin / Agency (web portal) | Backend | Device |
|---|---|---|---|
| 0 | Opens property page, picks sensor, clicks "Test" | — | Idle |
| 1 | `POST /users/v1/alarms/test-alarm` (alarmId, optional `scheduleTime`) | Looks up hub serial via `controllerId`, publishes `{"CMD":"TEST","INDEX":n}` to `sg/sas/cmd/<serial>`, sets Redis `<serial>_HUB_VERIFY` trigger flag | ← TEST |
| 2 | (waiting on socket) | — | Hub RF-fires sensor; sensor enters test mode, beeps |
| 3 | — | — | → `{"CMD":"TEST","INDEX":n,"STATUS":1\|2\|0}` |
| 4 | UI updates: pass / in-progress / fail | Writes `tbl_alarms.testStatus`, `lastTestedAt`; appends `AlarmLogs.testResults` | Idle |
| 5 | If `scheduleTime` was given, the whole sequence happens automatically at that time, otherwise immediate | (Scheduler / cron path — verify in code if relevant) | — |

### 3c. Periodic / unsolicited device events

| Event from device | Backend action | What admin/agency sees |
|---|---|---|
| `BATTERY` (periodic per-sensor %) | Updates `tbl_alarms.battery` | Battery column on portal |
| `LOW_BATTERY` at 15 % | Updates state, sends notification | Low-battery alert; raises maintenance need |
| `LOW_BATTERY` at 10 % / 5 % | **Silently dropped** (known gap, `alarm-event-traces.md:164`) | Nothing — re-alerts lost |
| `FAULT` (any code) | Logged | Visible in event history |
| `FAULT CODE 10` (cleaning required) | **No handler** (known gap, `alarm-event-traces.md:223`) | Nothing surfaced |
| Hub disconnect (MQTT keep-alive lapse) | `connectedStatus=DISCONNECTED` on hub row | Hub shown offline; no auto-recovery action |
| Hub reconnect | Hub re-subscribes; backend doesn't automatically re-VERIFY → see "index drift" risk below | Hub flips back to online; sensor inventory not re-checked unless manually triggered |

### 3d. Manual hub refresh (when something looks wrong)

| # | Admin / Agency | Backend | Device |
|---|---|---|---|
| 0 | Notices stale / missing sensor data on portal, clicks "Refresh hub" | Publishes `{"CMD":"VERIFY"}` then `{"CMD":"ALARMS"}` (same chain as install step 13) | ← VERIFY, ← ALARMS |
| 1 | — | `syncAlarmList()` updates INDEX→serial mapping | → VERIFY response, → `ALARMS` response |
| 2 | Inventory + index map refreshed; UI catches up | — | Idle |

This is the recovery path for **index drift** if pairing churn has occurred
since the last sync. (How often this is needed in practice is unclear — open
question.)

### 3e. Maintenance, replacement, removal jobs

These re-use the install flow tables above with a different job type. Verified
against `smokealarm-android/.../ApiConstants.kt` and `smokealarm-ios/.../EndPoints.swift`:

**Job types the app actually submits** (from `ConstantEnums.kt`):
`INSTALLATION(1)`, `MAINTENANCE(2)`, `ADD(3)` — there is **no** `REPLACEMENT`
job type. `REPLACEMENT_TYPES` (`WARRANTY` / `OWNER_COST`) is a cost-category
label, not a job type.

**Endpoints used in maintenance / replacement / removal:**

| Action | Endpoint | Method |
|---|---|---|
| Add hub or sensors | `users/v1/alarms` | `POST` |
| **Replace a sensor in place** | `users/v1/alarms/replace` | `PUT` |
| Remove a sensor | `users/v1/alarms/<id>` | `DELETE` |
| **Remove a hub (controller)** | `users/v1/alarms/remove-controller` | `PUT` |
| Re-sync hub/alarms after error | `users/v1/alarms/retry-installation` | (see notes) |
| Test a sensor | `users/v1/alarms/test-alarm` | `POST` |
| Start a job | `users/v1/jobs/update-start-time` | `PATCH` |
| Complete a job | `users/v1/jobs/complete` | `PUT` |

**Practical implications:**

- **MAINTENANCE** — installer visits, may run TEST / BATTERY refresh; uses
  `PUT /users/v1/alarms/replace` for **sensor swap in place** (NOT remove +
  add). This is a single call: backend handles the REMOVE-then-ADD MQTT chain
  server-side and reassigns the row to the new serial. Earlier draft of this
  doc that described replacement as "remove then add" via the app was wrong.
- **REMOVAL** — sensors via `DELETE /users/v1/alarms/<id>`; hub via
  `PUT /users/v1/alarms/remove-controller` (distinct endpoint, distinct
  method). Doc previously generalised both as "DELETE" — that's wrong for the
  hub.
- **INSPECTION** — on-demand TEST sweep (3b) repeated for every sensor; no
  device-state mutation.
- **`retry-installation`** endpoint exists in both apps' API constants but
  whether the installer UI exposes a button for it is **unverified** — the
  agent searching didn't find a UI affordance. Likely used internally on
  failure paths or only from the admin portal.

The lifecycle (PENDING → ACCEPTED → IN_PROGRESS → COMPLETED) and the actor
swim-lanes are identical to the install case — only the API verb at the
"touch the device" step differs (add vs replace vs remove vs test-only).

**Revised sensor-replacement sequence (single-call, in-place):**

| # | Installer (native app) | Backend | Device |
|---|---|---|---|
| 0 | Picks faulty sensor → "Replace" with new serial: `PUT /users/v1/alarms/replace` | — | Hub online; old sensor possibly faulty |
| 1 | (waiting) | Backend issues `{"CMD":"REMOVE","ALARMSERIAL":"old"}` then `{"CMD":"ADD","ALARMSERIAL":"new"}` on the same hub | ← REMOVE → response; ← ADD → response |
| 2 | UI flips sensor row to new serial, same location, refreshed INDEX | `syncAlarmList` (VERIFY → ALARMS) reconciles INDEX→serial | → ALARMS list with new mapping |
| 3 | Taps **Test** on the new sensor | publishes `TEST` | beep → STATUS |
| 4 | Notes + photos → **Complete** | Job → COMPLETED | Idle |

This is what the app actually does. The previous A-table (manual
remove-then-add over two API calls) describes a path the installer **could**
take if `replace` failed or wasn't available, but it isn't the default.

**Hub replacement** still requires the multi-step path in the previous B
table — there is **no** `replace-controller` endpoint, only the
`PUT remove-controller` + `POST /alarms` (new hub) + N × `POST /alarms` (sensors)
sequence. So B is unchanged structurally; only correct the hub-removal verb
from `DELETE` to `PUT /users/v1/alarms/remove-controller`.

### Reading notes for post-install

- **Most traffic is device → backend → agency**, not the other way around.
  The portal is almost entirely a read/observe surface plus the occasional
  TEST trigger.
- **Installer is offline** during steady-state. They only re-enter the
  picture when a new job is created (sec 2, step 0).
- **Known silent-drop gaps** (FAULT 10, LOW_BATTERY 10 %/5 %, STATE:2 alert
  clear) are post-install reliability risks worth tracking — they don't
  affect the install ceremony but degrade ongoing compliance signal.
- **Hub never initiates pairing** even after a reconnect. If a sensor was
  swapped or fell off RF while the hub was offline, the backend won't know
  until someone (admin or installer) issues a manual VERIFY → ALARMS.

---

## 4. Detach / return to stock

Triggered when equipment is taken off a property (faulty hub, end-of-tenancy
removal, redeployment to another site). Code-grounded as of 2026-05-21.

**Where it lives:** **admin portal only.** The native installer app has the
`remove-controller` endpoint constant compiled in
(`smokealarm-android/.../ApiConstants.kt:44`,
`smokealarm-ios/.../EndPoints.swift:124`) but **no UI affordance**. On-site
field staff complete a MAINTENANCE/INSPECTION job in the app; the actual
detach is run separately from the office. Coordination is manual.

**Single entry point:** `PUT /users/v1/alarms/remove-controller` — same
endpoint hit by both the admin portal (`sensor-angular`
`alarms-list.component.ts:320-349` via URL constant `urls.ts:128`) and any
direct mobile-app call. Handler at `sensor-alarm-backend/src/controllers/
alarms.controller.ts:624-873`.

### 4a. Detach swim-lane

| # | Admin / Agency (web portal) | Backend | Device (hub + sensors, MQTT) |
|---|---|---|---|
| 0 | Opens hub row in property page; clicks delete; confirms dialog | — | Hub may be online or offline |
| 1 | `PUT /admins/alarms/remove-controller` (`{Id}`) | Loads hub row (`Alarms.findOne(id) where controller=1`) | — |
| 2 | (waiting) | Soft-deletes children: `tbl_alarms set status=DELETED, connectedStatus=REMOVED where controllerId=<hubId>` (`controllers/alarms.controller.ts:729-738`) | — |
| 3 | (waiting) | Unlinks hub row: `propertyId=NULL, jobId=NULL, location=NULL, connectedStatus=REMOVED, simStatus=inactive` — **but `status=ACTIVE`** so it remains queryable as available stock (lines 710-717) | — |
| 4 | (waiting) | Clears open alerts: `tbl_alarmAlerts.eventStatus=false` for hub + all children (lines 742-752). Rows kept for history | — |
| 5 | (waiting) | Marks property: `property.status=DISCONNECTED, alarmStatus=INACTIVE` (lines 754-768) | — |
| 6 | (waiting) | Appends removed sensor serials to `tbl_jobs.removedSerialNumbers` (line 799). **No matching log row for the hub itself** | — |
| 7 | (waiting) | Calls **KORE** SIM API → SIM `INACTIVE` (lines 831-841). **Stale code in Omondo era — see notes** | — |
| 8 | (waiting) | Publishes `{"CMD":"RESET"}` to `sg/sas/cmd/<hub-serial>` (line 828) — fire-and-forget, no ack | ← `RESET` (if online) |
| 9 | Toast: "SENSOR_HUB DELETE_SUCCESS" | Returns success regardless of MQTT outcome | If online: factory-resets, drops all RF pairings. If offline: nothing — hub still believes it's paired |

### 4b. Implicit "stock" state

There is **no `stockStatus` column, no inventory table, no warehouse concept**
in the schema (`models/mysql/alarms.ts`). A hub is in "stock" iff:

```
controller = 1
AND propertyId IS NULL
AND status = ACTIVE
AND connectedStatus = REMOVED
```

This is the only marker.

### 4c. Re-deployment to another property

`installationJob` (`controllers/alarms.controller.ts:589-733`) handles re-use:

| Condition on the existing row | Behaviour |
|---|---|
| `serialNumber` not found | INSERT new row |
| Found, `propertyId IS NULL` | UPDATE in place: new `propertyId`, `jobId`, `location`, re-activate SIM |
| Found, `propertyId IS NOT NULL` | Reject with `HUB_ALREADY_INSTALLED_ON_PROPERTY` (lines 603-619) |

So the same physical hub can be installed on Property B after detach, with
the same DB row and full history continuity.

### 4d. Single-sensor detach (not a full hub detach)

`DELETE /users/v1/alarms/<id>` → `alarmEntity.removeAlarmFromAProperty`
(`entities/alarms.entity.ts:986-1057`). Per-sensor flow:

- Sensor row: `status=DELETED, connectedStatus=REMOVED` (line 1051). Row kept.
- Sensor's open `AlarmAlerts` cleared (lines 997-1007).
- Serial appended to `jobDetails.removedSerialNumbers` (lines 1020-1025).
- Mongo `AlarmLogsModel` historical entries untouched.
- Backend publishes `{"CMD":"REMOVE","ALARMSERIAL":...}` to the hub so the
  hub clears the RF pairing for that one sensor.

This is the right path for "I want one bad sensor back, hub stays in place."
For full equipment-return, use `remove-controller` (4a) which cascades.

### 4e. Gaps to flag

- **Offline-hub detach is unsafe.** The DB is updated synchronously and the
  call returns success even if the `RESET` MQTT publish lands on nothing. The
  physical hub, when next powered up, still believes it's paired to the old
  sensor inventory. Next installer plugging it into Property B sees ghost RF
  pairings with no warning. There is no retry queue, no delivery
  confirmation, no "pending reset" state on the row.
- **No "stock" semantics.** A hub with `propertyId=NULL, status=ACTIVE` is
  indistinguishable from a half-completed install or a row in transit. No
  way to mark `RETIRED`, `DEFECTIVE`, `IN_WAREHOUSE`, `LOST`.
- **No on-site detach affordance.** Field installers can't return equipment
  themselves from the native app; an office user has to follow up via the
  portal. This is a process gap that the new safer-ops kit flow will need
  to address too.
- **SIM lifecycle code is stale.** The KORE deactivation call at lines
  831-841 was for the previous SIM provider. The fleet today uses **Omondo
  pre-activated SIMs** (see project memory `sim-provider-omondo`); this call
  is either silently failing or being short-circuited in prod. Worth an
  audit — if it throws, does it abort the whole removal?
- **Asymmetric audit trail.** Each removed sensor gets a `tbl_alarmLogs`
  entry; the controller removal itself does not. Reconstructing "who
  detached this hub when" relies on `tbl_jobs.removedSerialNumbers` + portal
  access logs.
- **`{"CMD":"RESET"}` is the hammer.** No selective unpair at controller
  level — only per-sensor REMOVE during a maintenance job before the
  controller delete. Fine for return-to-stock, awkward for partial detach.

### 4f. Open questions

- Does the KORE SIM call currently error in prod (Omondo era)? Is the error
  caught silently, or does it abort the removal flow?
- Is `{"CMD":"RESET"}` reliably honoured by hub firmware across power cycles
  and stale-pairing state? Tied to the firmware-retry open question.
- How many prod hubs are currently sitting in implicit-stock state
  (`controller=1 AND propertyId IS NULL AND status=ACTIVE`)? A count would
  show whether this path is used routinely or whether equipment moves between
  properties without ever formally detaching.

---

## 5. Verification log (2026-05-21)

Each claim above was cross-checked against the native installer apps
(`smokealarm-android`, `smokealarm-ios`) in addition to the backend.

**Confirmed against native app (✅):**
- `GET users/v1/jobs/app` — installer job list. Android `ApiConstants.kt:105`,
  iOS `EndPoints.swift:105`.
- Hub registration `POST users/v1/alarms` (controllerData). Android
  `ApiInterface.kt:315`, iOS `APICaller+CreateJob.swift:167`.
- Sensor registration — same endpoint, `alarmsData[]`. Android
  `ApiInterface.kt:320`, iOS `APICaller+CreateJob.swift:182`.
- Test: `POST users/v1/alarms/test-alarm`. Android `ApiConstants.kt:34`, iOS
  `EndPoints.swift:98`.
- Complete job: `PUT users/v1/jobs/complete`. Android `ApiConstants.kt:115`,
  iOS `EndPoints.swift:110`.
- Sensor delete: `DELETE users/v1/alarms/<id>`. Android `ApiInterface.kt:302`,
  iOS `APICaller+CreateJob.swift:210`.

**Corrected against native app (changes applied above):**
- Job start: now reflected as `PATCH users/v1/jobs/update-start-time`
  (Android `ApiConstants.kt:118`, iOS `APICaller+CreateJob.swift:345`) — was
  implicit before.
- Sensor replacement: now described as single `PUT users/v1/alarms/replace`
  call (Android `ApiConstants.kt:42`, iOS `EndPoints.swift:108`) — earlier
  draft incorrectly described remove-then-add.
- Hub removal: now `PUT users/v1/alarms/remove-controller` (Android
  `ApiConstants.kt:44`, iOS `EndPoints.swift:124`) — earlier draft said
  `DELETE`.
- Job types: only `INSTALLATION(1)`, `MAINTENANCE(2)`, `ADD(3)` from
  `ConstantEnums.kt:65-70` — no `REPLACEMENT` job type. Earlier draft
  implied otherwise.
- Sockets: only `sensor-angular` (admin portal) has a `SocketService` /
  `socketUrl` config. The native installer apps have no socket.io / websocket
  client — they rely on HTTP responses + FCM/APNs push. Doc text and table
  cells updated accordingly.

**Unverifiable / open (❓):**
- 10-second ADD timeout: implemented on the backend (per
  `alarm.controller.ts:182-193`); no matching client-side timer found in the
  apps. The push-notification surfacing is the assumed mechanism but not
  directly confirmed.
- `retry-installation` endpoint exists in app API constants but no UI
  affordance for it was located in the installer app — may be a programmatic
  fallback path only, or surfaced only in the admin portal.
- Sensor-side "press to reset" prompt during hub replacement: no UI flow
  found in either native app. Confirmed as tribal/operational knowledge,
  not in-product.
