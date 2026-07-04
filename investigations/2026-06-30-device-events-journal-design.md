# Independent device-event journal (`tbl_device_events`) — design

**Date:** 2026-06-30
**Status:** approved (capture mechanism revised to independent MQTT tap), implementation in progress
**Scope:** new standalone MQTT consumer + Mongo `sensorproddb.tbl_device_events`

## Problem

`tbl_alarm_logs` (Mongo) is the only replayable record of device behaviour, but it is
**not append-only** and it is **Sensor's interpretation**, not ground truth. The same
physical document is rewritten in place by several paths (6-min timeout sweep
`alarm.controller.ts:1476`; in-place TEST overwrite `subscriber.ts:3168`; location
back-fill `alarms.controller.ts:2208`; flapping/alert/cert flags
`subscriber.ts:6090+`, `jobLogs.entity.ts:2529`). `timestamps:true` + a `beforeEdit`
field confirm in-place mutation by design. The document fuses the immutable event,
retroactively-edited "facts", and mutable workflow state — so a row read at *T* may say
something different at *T+6min*, and anything Sensor mis-decodes or never processes
(because it was down or buggy) is simply absent or wrong.

## Decision

Capture **ground truth off the wire**, independently of `sensor-alarm-backend`, into a
separate **append-only** journal — `tbl_device_events` — usable for forensics and to
**augment** (not replace) Sensor's view. A standalone MQTT consumer mirror-subscribes the
broker, records exactly what each device emitted and what commands were sent, frozen at
capture time, and never mutated. Sensor's *derived* signals (the "Hub did not respond"
timeout verdict, DISCONNECT/RECONNECT, ALERT) stay in `tbl_alarm_logs`; forensics
cross-references the two — and any divergence between them is itself a finding (a Sensor
decode/ingest bug caught by an independent witness).

This complements the MySQL `tbl_device_operation_events` (`recordEvent()` in
`deviceOperations.entity.ts`), which is append-only but scoped to bounded
pre-paired/stock-verification operations.

### Why an independent tap (not a hook in Sensor)

- **Ground truth, surviving Sensor.** A wire tap records what actually happened even when
  Sensor is down, mis-deployed, or buggy; a hook inherits Sensor's mistakes and downtime.
- **Zero Sensor regression surface.** Touches no Sensor code — a pure new consumer. The
  whole "modify the shared `alarmLogsSchema`, regression-guard, behaviour-preserving,
  kill-switch" concern disappears.
- **Direction is deterministic, decoding is trivial.** See wire reality below.
- **Validated precedent.** `docs/investigations/2026-05-15-mqtt-lab-routing-design.md`
  already recommends an application-level mirror-only router (distinct clientId + distinct
  group); `scripts/diag/mqtt-diag.js` `listen` mode is a working read-only prototype of it.

### Wire reality (confirmed from `subscriber.ts` + `mqtt-diag.js` + the routing doc)

- Broker: **production EMQX**, username/password auth, **IP-allowlisted to the app nodes**.
- Topics: device→server `sg/sas/resp/<hubSerial>`; server→device `sg/sas/cmd/<hubSerial>`.
  **Direction = topic prefix; hubSerial = topic segment.** (`subscriber.ts:188` reads
  `topicArray[3]`.)
- Payload: **UTF-8 JSON** (`JSON.parse(buf)` → `{CMD, STATUS, BATTERY, INDEX, ALARMSERIAL,
  ALARMLIST, POWERSTATE, …}`). **No binary/firmware decoder to reimplement.** CMD
  vocabulary is documented in `code/mqtt-prototype/firmware.md`.
- Non-interference: the backend consumes via a **shared** subscription
  `$share/sensorGroup/sg/sas/resp/+` (`subscriber.ts:126`). A **plain** (non-`$share`)
  subscription with a **unique clientId** gets its own mirror copy and never steals a
  frame. **Hard rule: never join `$share/sensorGroup/...`.**

### Store: Mongo time-series collection (grounded in measured volume)

Measured 2026-06-30 on live `tbl_alarm_logs`: 589,629 docs / 91-day window ≈ **6,465/day**
(~2.36M/yr); **92.9% are VERIFY connection polls**; data 175.7 MB but **235.9 MB across 24
indexes**. Extrapolated firehose: ~1.6 GB/yr in a 24-index collection vs **~0.15–0.3 GB/yr
in a time-series collection** (columnar bucketing crushes the 93% near-identical frames).

Chosen: a **Mongo time-series collection**, full firehose incl. heartbeats. TS compression
makes heartbeats nearly free (preserving the heartbeat-*absence* signal that exposed the
dead Essex hub); native TTL gives retention for free; append-only + fixed
`metaField`/`timeField` matches the only query shape. Volume is far too small for a new
TSDB; MySQL rejected (93% heartbeats expensive in InnoDB; no relational join needed).

## Document shape (immutable)

- `ts` (Date) — **tap receive time** (broker delivery; frames carry no device timestamp).
- `meta: { hubSerial }` — the only identity known at capture; stable ⇒ safe TS metaField.
- `direction: "command" | "report"` — from the topic prefix (cmd vs resp). Deterministic.
- `topic` — the raw topic, verbatim.
- `event` — `CMD` from the payload (VERIFY/ADD/ALARMS/TEST/TAMPERED/BATTERY/POWER/…).
- `childSerial` (`ALARMSERIAL`), `index` (`INDEX`) — from payload when present.
- `status`/`outcome` — the device's own `STATUS`/`STATE`, frozen; never re-stamped.
- `battery`, `powerState`, `rawDetails` (the parsed JSON frame verbatim).
- `controllerId`, `propertyId`, `jobId` — **best-effort enrichment** (serial→entity MySQL
  lookup) done **before insert**, never as a later mutation; `null` when unresolved
  (resolve at read time). Kept as measurement fields, NOT in `meta`, so enrichment never
  rewrites a bucket key.
- `correlationId` — only obtainable via enrichment/operation join (not on the wire);
  nullable.
- `ingest: "tap" | "backfill"`, `backfilled` (bool), `backfillBatch`, `sourceId` (original
  `tbl_alarm_logs` `_id` for backfilled rows) — provenance.
- `schemaVersion`.

**Excluded:** all mutable workflow flags (`alertSent`, `flapping*`, `certificateCreated`,
`hideFromAlarmsLogs`) and Sensor's *derived* events (timeout/disconnect) — those stay in
`tbl_alarm_logs`.

## Indexes (~5)

TS auto-creates `(meta, ts)`. Plus: `{propertyId:1, ts:-1}` (per-property; measurement-
field index needs Mongo ≥6.3), `{"meta.hubSerial":1, ts:-1}` (per-hub), `{childSerial:1,
ts:-1}` (per-child), `{correlationId:1}`, `{testId:1}`. No unique indexes (TS constraint).

## Components

1. **Tap consumer (new standalone service).** Connect EMQX (creds from Secrets Manager),
   **stable dedicated clientId**, **`clean:false` + QoS 1**, auto-reconnect; plain-subscribe
   `sg/sas/resp/#` and `sg/sas/cmd/#`. On message: topic → `{direction, hubSerial}`;
   `JSON.parse(payload)` → frame; best-effort cached enrichment; buffer + `insertMany` to
   `tbl_device_events`. Capture never depends on MySQL being up (insert with `null`
   enrichment on miss).
2. **Storage.** `tbl_device_events` TS collection + indexes (created explicitly on prod).
3. **Backfill (deferred — not built).** A future one-off checkpointed runner over
   `tbl_alarm_logs` (live, then `MONGO_DB_ARCHIVED`) could project historical rows with
   `ingest:"backfill"`, `backfilled:true`, `sourceId`. It would be **standalone** (the
   earlier backend-side `src/utils/deviceEvent.projection.ts` was removed with the
   abandoned sensor-driven approach). Unlike live capture — where direction is read off the
   topic — the backfill must **reconstruct** `{direction, status, outcome}` from a mutated
   `tbl_alarm_logs` row, disambiguating three shapes:
   - *outbound command* — `details == null` && `status == REPORT_STATUS.PROGRESS(3)`; `event` is the CMD; no `eventTriggerSource`.
   - *device report* — `details` is an object + `eventTriggerSource` set; the device's real answer is `details.STATUS` (`STATUS == 2` on an ADD means "already bonded", NOT a failure).
   - *timeout-sweep stamp* — `details == null` && `status == REPORT_STATUS.FAIL(2)`, no `eventTriggerSource` (the late verdict on an unanswered command).
   Historical rows inherit `tbl_alarm_logs`' already-mutated state ⇒ explicitly best-effort;
   only `ingest:"tap"` rows are authoritative ground truth.

## Deployment & durability (the elevated decision — needs infra owner)

The broker is IP-allowlisted to the app nodes, so the tap must run inside that boundary or
have its source IP added to the EMQX allowlist (a networking/security-group change ⇒
elevated, name-the-resource approval).

- **Durability is bounded by the publishers' QoS.** Backend command publishes and device
  frames are QoS 0; EMQX does not queue QoS-0 messages for an offline persistent session.
  So a `clean:false` + QoS-1 *subscription* helps only marginally; the practical guarantee
  is **at-most-once: captured while connected, with gaps during tap/broker outages.** For
  forensics this is usually acceptable — maximise uptime (auto-reconnect, fast restart,
  optionally HA/redundant taps deduped by `(topic, ts, payload)`).
- **Highest-durability alternative:** an **EMQX server-side rule / data-integration** sink
  (every `sg/sas/{resp,cmd}/#` message persisted broker-side to Kafka/SQS/Mongo, no client
  uptime dependency). Most reliable, but it's an EMQX config change (elevated).

Deployment options to choose: (A) dedicated in-VPC service (ECS Fargate / EKS pod) in the
app security group; (B) EMQX server-side rule → durable sink (most reliable, no client);
(C) cheap start — a `systemd`/pm2 process co-located on an existing app node (already
allowlisted; fastest, not HA).

## Rollout (mutations gated propose → go; elevated infra named; record in `docs/applied-changes.md`)

0. This doc.
1. **Storage** — confirm Atlas ≥5.0 (≥6.3 for measurement-field indexes); `createCollection
   tbl_device_events` with TS + TTL + indexes (propose → go).
2. **Tap service** — build the standalone consumer (pure addition, no Sensor code). Validate
   off-prod / read-only first (the `mqtt-diag listen` path already proves connectivity).
3. **Deploy** — place the tap per the chosen deployment option (elevated: name the
   resource — SG/allowlist or EMQX rule). Run read-only/dark, reconcile a window against
   `tbl_alarm_logs` device frames; measure compression.
4. **Backfill** — checkpointed historical load with the `backfilled` indicator.
5. **(Optional)** read-time forensic/narrative tool over `tbl_device_events`, cross-linked
   to `tbl_alarm_logs`.

Config: `DEVICE_EVENTS_TTL_DAYS`, broker creds (existing `MQTT_*` secret), tap clientId.
