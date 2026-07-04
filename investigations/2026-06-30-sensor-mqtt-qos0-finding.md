# Finding: Sensor backend subscribes to device MQTT at QoS 0 (reliability gap)

**Date:** 2026-06-30
**Status:** parked — measure via the forensic tap before any change
**Severity:** medium-high (life-safety platform), but change is high-risk (Class C)

## Finding

`sensor-alarm-backend` consumes device frames via a **QoS 0** shared subscription —
`mqttClient.subscribe("$share/sensorGroup/sg/sas/resp/+", { qos: 0 })`
(`src/services/mqtt/subscriber.ts:126`) — and publishes commands at mqtt.js default QoS 0
(`sendMessage`/`sendMessageCron`, `subscriber.ts:759-775`).

Measured on prod EMQX `i-0b381a2bfdf65a329` (open-source 5.8.7), 2026-06-30:

| Inbound publishes (device/backend → broker) | share |
|---|---|
| QoS 0 | 19,334 (82.9%) |
| QoS 1 | 0 |
| **QoS 2** | **3,993 (17.1%)** |

| Outbound deliveries (broker → subscribers) | share |
|---|---|
| QoS 0 | 29,456 (100%) |
| QoS 1 / 2 | 0 |

`durable_subscriptions.count = 0` (no persistent sessions in use).

## Why it matters

`min(publishQoS, subscriptionQoS)` governs delivery. ~17% of device publishes are **QoS 2
(exactly-once)** — the firmware deliberately marks those frames critical — but the
backend's QoS-0 subscription **downgrades every one to QoS 0 on delivery** (hence
qos2.sent = 0). QoS 0 = no broker storage, no retransmit: a frame in flight during a
backend reconnect (ASG deploy/scale), a busy GC pause, or a broker blip is **silently
dropped**, and a QoS-0 subscriber cannot detect the loss. For a smoke-alarm platform,
silently dropping the frames the firmware flagged as critical is a real reliability gap.

## Why it is NOT a quick flip

Raising the subscription to QoS 1/2 is a **Class C** change to live Sensor MQTT behaviour.
QoS ≥1 introduces **redelivery**; if the subscriber handlers are not idempotent,
redelivery causes **duplicate side effects** — double bonding, duplicate alert
emails/SMS, duplicate DB writes. A QoS change must be preceded by a handler-idempotency
audit and (likely) a dedup layer.

## Recommended sequencing

1. **Do not change Sensor now.**
2. Stand up the forensic tap at **QoS 2 + persistent session** (separate workstream).
3. Use the tap to **measure the real drop rate** — frames the tap captures that
   `tbl_alarm_logs` never recorded — to quantify the gap.
4. If material: idempotency-audit the subscriber handlers, then raise the subscription QoS
   (QoS 1 broadly, or QoS 2 for critical topics) with dedup — propose → go, regression-guard.

This finding is the motivation; the tap provides the evidence.

## Measured via the tap — 6-hour clean window (2026-07-01, 00:37–06:35 UTC)

Ground-truth reconciliation: tap `tbl_device_events` **reports** (device→server, captured at
QoS 2) vs `tbl_alarm_logs` device-originated rows (`eventTriggerSource ∈ {HUB, HEARTBEAT}`),
identical window. 3,214 total tap events; 429 of them inbound reports (the rest are outbound
`command` mirrors — VERIFY polls etc. — not governed by the subscription QoS).

| Event (device→server) | Tap (QoS-2 truth) | Sensor logged | Δ |
|---|---|---|---|
| BATTERY | 116 | 116 | **0** |
| VERIFY (HUB) | 82 | 82 | **0** |
| TEST | 36 | 36 | **0** |
| POWER | 12 | 12 | **0** |
| ALERT (fire/smoke) | 2 | 2 | **0** |
| REMOVE | 3 | 3 | **0** |
| TAMPERED | 65 | 64 | 1 |
| ALARMS | 60 | 58 | 2 |
| ADD | 15 | 12 | 3 |
| LOW_BATTERY | 38 | 0 | 38 (code discard, not QoS) |

**The control that makes this conclusive: ZERO backend MQTT reconnects in the window.**
CloudWatch `smoke-api-prod-pm2-out-log` shows no `mqttClient Connected`/`Reconnecting` lines
across 00:30–06:40 (the backend logs both on `subscriber.ts:124/133`). The one "reconnect"
in logs is a Mongo blip at 00:31; the `RECONNECT`/"Reconnection Advice" rows are *device*
hubs coming back online, not the backend client. The subscription held one continuous session
the whole window.

**Result: at steady state (no disruption), QoS-0 is effectively lossless** — exact parity on
every safety-critical type (ALERT 2/2, TEST 36/36, POWER 12/12, BATTERY 116/116). With the
session continuously established, QoS-0 over the underlying TCP stream cannot drop in-order
frames, so this is exactly what theory predicts. This **refutes** the earlier "entire
device-report stream is at risk" framing.

**The residual Δs are NOT transit loss** (a zero-reconnect window definitionally can't lose
QoS-0 frames) — they are logging-path differences:
- **ADD 15 vs 12** — all 15 tap ADD reports are `outcome:1` (success); the 3 not in
  `tbl_alarm_logs` are ADD frames inside pairing **operations**, recorded in the MySQL
  operation-event log (`tbl_device_operation_events`, `recordEvent()`), not the alarm log.
- **ALARMS 60 vs 58 / TAMPERED 65 vs 64** — heartbeat-sweep list-reply dedup and window-edge
  boundary (tap `ts` = receive time vs alarm-log `createdAt` = process time). Sub-frame,
  non-safety-critical.
- **LOW_BATTERY 38 vs 0** — the code discard at `subscriber.ts:197`
  (`2026-07-01-sensor-discards-low-battery-frames.md`), unrelated to QoS.

**So the QoS-0 *transit drop* still has no observed instance** — because the one thing that
causes it (a reconnect) did not happen in any window captured so far. The risk is therefore
**entirely concentrated in disruption windows**; see the exposure model below.

## Potential impact — where the QoS-0 risk actually lives

Steady state is safe; the exposure is the **reconnect gap**. The backend runs a `$share`
group across **2 API nodes** (`prod-api-server` ×2), each carrying ~half the fleet's frames.
On any deploy / ASG scale / node bounce, the affected node disconnects and re-subscribes
(`reconnectPeriod: 1000ms`; a golden-AMI rollout gap is seconds→tens of seconds). Two loss
modes bite **only** in that gap:

1. **In-flight QoS-0 frames to the reconnecting member are dropped, not rerouted.** At QoS 0
   the shared subscription does **not** re-dispatch to a surviving group member (QoS≥1 would).
   So ~half the fleet's inbound frames during one node's gap are exposed.
2. **A QoS-0 subscriber cannot detect the loss** — no gap counter, no redelivery, silent.

**What is genuinely at risk vs self-healing:**
- **Self-healing (low impact):** VERIFY/BATTERY/ALARMS are **poll replies** — pull-driven. A
  dropped reply just looks like one missed heartbeat and is re-polled next sweep. The bulk of
  inbound volume falls here. Likewise, dropped *outbound* commands (Sensor→device, also QoS 0)
  self-correct via the 6-min "Hub did not respond" timeout re-poll.
- **Unrecoverable (the real risk):** spontaneous device **push** events — **fire/smoke ALERT**,
  TAMPERED, POWER-loss, spontaneous ADD/REMOVE. There is no re-poll for these; if one lands in
  a reconnect gap it is lost silently and forever. Measured push rate is low
  (ALERT ≈ 2/6h fleet-wide, POWER ≈ 12/6h, TAMPERED ≈ 65/6h), so per single reconnect the
  expected loss is a fraction of a frame and fire-ALERT loss is ~0.1%-order. **But**: reconnects
  recur on every deploy/scale, the loss is silent and unverifiable, and the one frame that
  matters most on a life-safety platform (a fire ALERT) is in the exposed class. Carrying an
  undetectable, unrecoverable loss path for fire alerts — however low the per-event odds — is
  the wrong risk to hold silently.

**Bottom line:** QoS-0 is not a steady-state firehose leak (measured ~exact parity). It is a
**low-probability, silent, unrecoverable tail risk on spontaneous safety-critical push frames
during backend reconnects.** That profile — rare but catastrophic-when-it-hits and invisible —
is exactly why it still warrants the QoS≥1 + idempotency remediation, not a shrug.

## Command (publish) side: server→hub under intermittent connectivity

Measured on prod EMQX `i-0b381a2bfdf65a329`, 2026-07-01 (`emqx ctl clients/subscriptions`):

- Each hub connects with `clientid = <hubSerial>`, **`clean_start=true`,
  `session_expiry_interval=0`**, and holds **one** subscription: its own command topic
  `sg/sas/cmd/<hubSerial>` **at `qos:2`**.
- So the firmware *asks for exactly-once command delivery* (subscribe QoS 2), but Sensor
  **publishes commands at QoS 0** (`sendMessage`, mqtt.js default). Effective delivery =
  `min(0, 2) = 0`. The device's reliability request is discarded at the publisher.

**What happens to a command sent while a hub is in a 4G dropout (the remote-area case):**

1. Sensor publishes to `sg/sas/cmd/<serial>` at QoS 0 — fire-and-forget, no broker storage.
2. The broker relays QoS 0 **only to a currently-connected session**. The hub's
   `clean_start=true` + `expiry=0` mean its session and subscription were **discarded the
   instant its 4G dropped** — there is *nothing to queue into*. The command is **dropped at
   the broker, silently.**
3. The QoS-2 subscription cannot save it (capped at 0 by the publish), and **even raising the
   publish to QoS 2 would not queue it** — a clean session with expiry 0 has no offline
   mailbox. Offline command-queuing would require a **firmware** change (persistent session:
   `clean_start=false` + non-zero `session_expiry`), which is a device decision, not a
   server flip. What the server *can* do unilaterally (raise publish QoS) only helps the
   *connected-but-lossy* transient, not the *hub-is-offline* window.

**So there is no messaging-layer self-heal.** "Self-heal" is entirely application-layer, and
only covers **re-pollable state**:

- **Poll commands (VERIFY/BATTERY/ALARMS):** genuinely self-heal. No reply within ~6 min →
  "Hub did not respond" (timeout sweep, `alarm.controller.ts:1476`) → re-issued next periodic
  sweep. The hub never needed the *lost* command — it just needs to be online for *some
  future* poll. Cost = staleness + a possibly-correct "DISCONNECTED" flag during the dropout.
- **One-shot commands (operator TEST, ADD/REMOVE during install):** **not** auto-healed — not
  on a sweep. Sensor shows "Hub did not respond"; a human retries until one lands in a
  connected window. (This is the already-known "test <5 min after boot fails" pattern.)
- **Momentary device→server events (fire ALERT, TAMPERED), the *other* direction:** no
  self-heal is even possible — while the hub is offline it cannot publish its uplink, and a
  fire at 02:14 is a point in time with nothing to re-poll. This is the physical limit: **an
  offline remote hub is a monitoring blind spot no server-side QoS can close.**

**The conceptual takeaway:** self-heal is not a delivery guarantee — it is Sensor
**re-deriving current state by re-polling**, so it works only for *state that persists and can
be re-read* (liveness, battery), never for *momentary events* (fire, tamper, a one-shot
command). QoS 0 makes both legs lossy, but for a flaky-4G hub the dominant problem is the
**offline window itself**; only the re-pollable state recovers when the hub returns.

## Next

- **Catch a real reconnect.** The tap now runs continuously; the next backend deploy / ASG
  event will produce the first *disruption* window. Re-run this reconciliation across it to
  put an actual number on gap-loss (expect a small, non-zero drop clustered at the
  `mqttClient Reconnecting` timestamp).
- The remediation path is unchanged (see "Why it is NOT a quick flip"): idempotency-audit the
  subscriber handlers, then raise the subscription to QoS 1 (QoS 2 for the critical topics)
  with a dedup layer — propose → go, regression-guard. The measured self-healing of poll
  replies means the idempotency surface that matters is the **push** handlers (ALERT/TAMPERED/
  POWER/ADD/REMOVE), which narrows the audit.
