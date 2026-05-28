# Kafka Message Flow Investigation

**Date:** 2026-04-12  
**Scope:** `sensor-alarm-backend` — Kafka producer and consumer topology, message types, data flow  
**Method:** Codebase analysis (`src/services/kafka/`, controllers, entities)

---

## Architecture Overview

Kafka is used exclusively within `sensor-alarm-backend`. No other service repo uses Kafka. The broker hosts a single topic: `logs-production` (pattern: `logs-${ENV}`).

```
┌─────────────────────────────────────────────────────────┐
│                  sensor-alarm-backend                   │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │              PRODUCERS (30+ sites)              │   │
│  │                                                 │   │
│  │  controllers/users/users.controller.ts          │   │
│  │  controllers/users/jobs.controller.ts           │   │
│  │  controllers/users/properties.controller.ts     │   │
│  │  controllers/users/settings.controller.ts       │   │
│  │  controllers/users/lease.controller.ts          │   │
│  │  controllers/users/accounts.controller.ts       │   │
│  │  controllers/users/roleAccess.controller.ts     │   │
│  │  controllers/invoiceSetting.controller.ts       │   │
│  │  controllers/sso.controller.ts                  │   │
│  │  controllers/jobs.controller.ts                 │   │
│  │  controllers/properties.controller.ts           │   │
│  │  entities/properties.entity.ts                  │   │
│  │  entities/jobs.entity.ts  + ~20 more            │   │
│  │                         │                       │   │
│  │          kafkaProducer.producer("logs", msg)     │   │
│  └─────────────────────────┼───────────────────────┘   │
│                             ▼                           │
│              ┌──────────────────────────┐               │
│              │   Kafka topic:           │               │
│              │   logs-production        │               │
│              │   (1 partition, 7-day    │               │
│              │    retention, 236 MB)    │               │
│              └──────────────┬───────────┘               │
│                             │                           │
│  ┌─────────────────────────▼───────────────────────┐   │
│  │  CONSUMER GROUP "smoke-alarm" (5 API instances) │   │
│  │                                                 │   │
│  │  consumer.ts — eachMessage handler              │   │
│  │                                                 │   │
│  │  msg.type == "subscription" ──►  invoiceSetting │   │
│  │                                 .checkSubscription│  │
│  │  msg.type == "inviteLandlord" ─► AdminProperties│   │
│  │                                 .landlordInvitation│  │
│  │  else ──────────────────────── ► logsEntity.add │   │
│  │                                   ├── LOG_TYPES.JOB     → MongoDB JobLogsModel       │
│  │                                   ├── LOG_TYPES.ALARM   → MongoDB AlarmLogsModel     │
│  │                                   ├── LOG_TYPES.TICKET  → MongoDB TicketModel        │
│  │                                   └── else             → MongoDB CommonLogsModel     │
│  └─────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────┘

MQTT subscriber (subscriber.ts) calls logsEntity.add() DIRECTLY — NOT via Kafka
```

---

## Message Types

The consumer handles three distinct message shapes on the single `logs-production` topic:

### 1. Log messages (the vast majority — ~289 call sites)

All API controllers and entities publish audit/activity logs using a shared payload shape:

```json
{
  "type": 1,           // LOG_TYPES enum: JOB=1, ALARM=2, PROPERTY=3, GENERAL=4, TICKET=5, WORKFLOW=6
  "action": 1,         // LOG_ACTIONS enum: CREATED=1, REJECTED=2, DELETED=3, ACCEPTED=4, ...
  "logType": "Action", // REPORT_TYPES: "Action" | "Email" | "Push" | "Sms"
  "id": <entityId>,
  "title": "...",
  "description": "...",
  "createdBy": <userId>,
  "receiverId": <userId>,
  "newChanges": { ... }
}
```

**Consumer action:** Routes to `logsEntity.add(msg)` which fans out to the appropriate MongoDB model:

| `msg.type` | MongoDB collection | Notes |
|---|---|---|
| `LOG_TYPES.JOB` (1) | `JobLogsModel` | Job lifecycle events |
| `LOG_TYPES.ALARM` (2) | `AlarmLogsModel` / `AlarmLogsModelArchived` | Alarm state changes |
| `LOG_TYPES.TICKET` (5) | `TicketModel` | Support tickets |
| All others | `CommonLogsModel` / `CommonLogsModelArchived` | General audit trail |

---

### 2. Subscription messages

```json
{
  "type": "subscription",
  "suspend": true | false,
  "status": "SUSPENDED",
  "propertyDetails": { ... },
  "subscription": { ... }
}
```

**Producer:** `invoiceSettingController.checkSubscriptionsStatus()` — triggered via an HTTP route (not a cron — it's called when an invoice/payment endpoint is hit).

**Consumer action:**
- `suspend: false` → `invoiceSettingController.checkSubscriptionAndUpdateStatus(subscription, propertyDetails)` — reactivates subscription
- `suspend: true` → `invoiceSettingController.cancelSubscriptionOnProperties(msg)` — suspends properties when payment lapses

**Purpose:** Decouples the payment webhook handler (fast HTTP response) from the subscription state change logic (slow DB writes across multiple properties). The Kafka message acts as an async task queue.

---

### 3. Landlord invitation messages

```json
{
  "type": "inviteLandlord",
  "allProperties": [ ... ],
  "sessionData": { ... },
  "params": { ... }
}
```

**Producer:** `KAFKA_ACTION_TYPE.INVITE_LANDLORD` is defined as a constant in `app.ts` but no direct `kafkaProducer.producer(... inviteLandlord ...)` call was found in the analysed source — the type may be produced by a legacy path or this consumer handler was written speculatively. The consumer handler has a `console.log("calling kafka msg", msg)` debug line, suggesting this path is not heavily used in production.

**Consumer action:** `AdminPropertiesClass.landlordInvitation(allProperties, sessionData, params)` — sends email invitations to landlords for multiple properties in batch.

---

## MQTT Events — NOT via Kafka

A common assumption would be that device MQTT events (alarm triggers, connections, battery alerts) flow through Kafka. **They do not.**

`src/services/mqtt/subscriber.ts` handles MQTT messages directly and calls `logsEntity.add(alarmLog)` synchronously on every API server instance. This means:

- All 5 API servers are simultaneously subscribed to MQTT and all 5 write alarm logs to MongoDB when a device event arrives
- There is **no Kafka mediation** for the alarm event path
- The alarm log writes are direct, synchronous (awaited) MongoDB inserts

This is relevant to the SMS drop investigation: the `tbl_alarm_alerts` MySQL writes (which drive SMS) also happen inside `subscriber.ts` via `alarmEntity`, not via Kafka.

---

## Key Architecture Observations

### Kafka is used as an async audit log queue, not for core alarm events

| Path | Via Kafka? | Notes |
|---|---|---|
| Alarm events (MQTT → SMS notification) | **No** | Direct MQTT subscriber → MySQL + MongoDB |
| Job lifecycle audit logs | **Yes** | Controller → Kafka → MongoDB |
| User/property management audit | **Yes** | Controller → Kafka → MongoDB |
| Subscription state changes | **Yes** | Payment webhook → Kafka → DB update |
| Landlord email invitations (bulk) | **Yes** (partially) | Kafka used to offload bulk email fan-out |

### Consumer group with 5 members = competing consumers, not fan-out

All 5 API instances share consumer group `smoke-alarm`. Each Kafka message is consumed by exactly **one** of the 5 instances (not all 5). This is correct for the log ingestion pattern — you don't want 5 instances all writing the same log entry to MongoDB.

However, the `logs-production` topic has **1 partition**. With 1 partition and 5 consumers in the same group, **only 1 consumer is ever active at a time** — the other 4 are idle standby. This is by design but limits throughput to a single consumer.

### `fromBeginning: true` + consumer clientId includes `iteration-4`

```ts
clientId: APP.KAFKA.CLIENT_ID + "iteration-4"
```
The `iteration-4` suffix in the client ID, combined with `fromBeginning: true`, suggests that historically the consumer group was periodically reset by bumping the iteration suffix to replay old messages. This is a fragile pattern and could cause double-processing of historical log messages if changed accidentally.

---

## Migration Implications

1. **Kafka is safe to cold-start** in new account — confirmed. Log messages are audit trails, not primary state. A brief gap during migration means some audit entries are missing but no operational data is lost.
2. **1 partition is sufficient** — the current throughput is low (audit logs from API calls). Keep 1 partition.
3. **Fix `log.dirs`** — new broker must use an EBS-backed path, not `/tmp`.
4. **Consumer group name `smoke-alarm`** must be reproduced in the new account (it's set via `KAFKA_GROUP_ID` env var in the secret).
5. **Client ID iteration** — do not change the `iteration-4` suffix during migration or it will replay all historical messages from the old broker (which won't exist anyway, but keep it stable).
6. **MQTT subscriber path is independent** — the alarm event + SMS path does not touch Kafka. EMQX migration is the critical path for device event reliability, not Kafka.
