# Installation Flow — Durability & Recovery Plan

**Date:** 2026-05-20
**Status:** Plan — not yet executed
**Scope:** `code/sensor-alarm-backend` (`deviceOperations`, branch `feature/additional-pairing-mode`) + `code/safer-ops` + a non-code firmware track.

## Context

The platform is moving smoke-alarm installation from on-site pairing to depot-prepared **kits**. In the new flow a kit (1 hub + N sensors) is pre-paired and tested in a depot, reserved to a job+property, and an installer uses the `safer-ops` web app on-site to scan the kit, re-verify it, and apply it via the sensor API `deviceOperations` endpoints. (Existing flow: see `diagrams/flow-device-installation.puml` and `infra/company-and-ownership.md`.)

The happy path is built and tested, but the flow has **no on-site failure recovery**. If a kit fails verification (sensor missing/dead, RF interference, hub unreachable), the `deviceOperations` operation goes `MISMATCHED` — which is terminal — and the installer is stuck. A smoke-alarm install can never be left half-done or ambiguous. This plan makes the flow durable and guarantees the installer always has a forward path.

## Decision (2026-05-20)

**Strict now, extensible later.** Phase 1: a kit install either fully matches (→ attach) or the installer falls back to the native app for the whole job. No partial installs via kit. The `deviceOperations` backend and kit states are designed so a future `attach-with-exceptions` (partial accept) path is a pure addition, not a rewrite.

## Goal & non-goals

**Goal:** the installer is never stuck — every on-site state has a forward path to a 100%-complete install (via kit or native fallback) or a clean abort that frees the job.

**Non-goals (phase 1):** partial installs; maintenance / replacement / add-product jobs via kits (these stay on the native flow); fully automatic re-pairing.

## The recovery ladder (phase 1, strict)

The installer web app walks the installer down this ladder and never shows a dead end:

1. **Retry** — re-run verification (transient RF flakiness).
2. **See the delta** — show exactly what is wrong (which sensor, which location), not "mismatch".
3. **Repair / swap** — scan a replacement sensor from spare stock, re-pair it, re-verify.
4. **Fall back to native app** — kit unsalvageable (hub dead, pairing lost): hand off to the native app with job + property pre-filled.
5. **Abort + release** — nothing works: mark the kit failed, release its reservation so the job can be re-dispatched with a new kit.

## Workstreams

### Phase 0 — Prerequisites: `deviceOperations` review fixes

The three High-severity findings from the `feature/additional-pairing-mode` review must land before the new flow drives production traffic:

1. **Fail-open the MQTT interception** — `src/services/mqtt/subscriber.ts`: `handleAlarmsResponse` / `handleDeviceOperationVerifyResponse` must catch their own errors and return `false`/`null`, so a thrown error never silently skips `syncAlarmList` (avoids a platform-wide alarm-sync outage).
2. **Sequential `bulkUpdate` under a transaction** — `src/entities/alarms.entity.ts` `installationJob`: when a `transaction` is passed, run the per-alarm chains sequentially, not via `Promise.all` (concurrent queries on one transaction connection are unsafe).
3. **Remove the debug log** in `sendInventoryMessage` (`subscriber.ts`).

### Phase 1 — Unstick the installer

#### Backend — `sensor-alarm-backend` / `deviceOperations`

- **Cancel/abandon path.** Add a `CANCELLED` status to `DEVICE_OPERATION_STATUS` (`src/constants/app.ts`) and an endpoint to cancel an in-flight operation (`AWAITING_VERIFY` / `AWAITING_ALARMS` / `MATCHED`). **Critical:** while an operation is in `AWAITING_*`, `subscriber.ts` intercepts the hub's `VERIFY`/`ALARMS` responses — so a native-app fallback install on the same hub would be hijacked. Falling back (ladder step 4) and aborting (step 5) **must** cancel the device-operation first. Files: `deviceOperations.entity.ts`, `deviceOperations.controller.ts`, `routes/users/v1/deviceOperations.routes.ts`.
- **TTL.** `OPERATION_TTL_MINUTES = 5` is too short for an installer working the recovery ladder. Refresh `expiresAt` on `VERIFY`/`ALARMS` activity and/or raise the TTL. File: `deviceOperations.entity.ts`.
- **Extensibility hook (no behaviour now).** Keep `MISMATCHED` operations retaining full `mismatchDetails` (`missingSerials` / `unexpectedSerials` / `missingIndexSerials`) and inspectable via `getOperation`; do not aggressively expire `MISMATCHED`. A future `attach-with-exceptions` endpoint then accepts a `MISMATCHED` operation + an acknowledged-exceptions payload — a pure addition.
- **Tests.** `deviceOperations` currently has none — add coverage for cancel, TTL refresh, and the mismatch → retry path.

#### `safer-ops` — installer web app + API

- **Recovery ladder UI** in the installer web app — on a mismatch, surface the `mismatchDetails` delta clearly and offer Retry / Repair / Switch-to-manual / Abort. Files: `apps/web/src/views/KitDetailPanel.tsx`, `CameraScanner.tsx`.
- **Kit recovery states** — add a `needs_attention` (recoverable) and a terminal `failed` state to the kit lifecycle; allow a `reserved`/attach-attempt kit to enter recovery and loop. Files: `apps/api/prisma/schema.prisma`, `apps/api/src/kits.ts`, `kit-rules.ts`.
- **Repair flow** — amend the kit's expected devices (swap a sensor), re-pair the replacement by **reusing the existing native pairing path** (do not rebuild pairing in `deviceOperations`), then re-verify via a fresh `pre-paired-verifications`. File: `apps/api/src/sensor-client.ts`, `kits.ts`.
- **Native fallback handoff** — hand off to the native app with job + property pre-filled; **must call the backend cancel endpoint first** so the device-operation does not hijack the native install.
- **Release reservation on abort** — free the job so it can be re-served with a new kit.
- **Wire `AuditEvent`** — the model exists but is unwired; log every recovery action so support can see why an installer got stuck.
- **Idempotency end-to-end** — use the existing `IdempotencyKey` model so a dropped connection mid-apply resumes instead of double-installing.
- **Offline states** — explicit "this step needs signal — queued, will retry" UX (verify/attach are online-only).

### Phase 2 — Durability hardening (prevent failures upstream)

- **Dispatch-time re-verification** — re-verify a kit before it leaves the depot (catches packing errors, confirms pairing immediately pre-transit). Add a dispatch/verify gate to the kit lifecycle.
- **Kit freshness checks** — flag stale pre-pairing / aged batteries before dispatch.
- **Reservation re-validation at scan** — re-check the job/property is still valid when the kit is scanned on-site (handles reschedule/cancel between reserve and install).
- **Observability** — surface `AuditEvent` + `simulatorTrace` / `lastMqttResponse` to support.

### Parallel track — Firmware confirmation (non-code, external)

Confirm with the firmware/hardware vendor whether RF pairing **persists through powered-off storage and transit**. The recovery ladder works either way, but this determines the expected on-site failure rate and whether to later invest in automatic re-pairing ("auto-converge"). Owner: third-party firmware vendor — needs an introduction (`gaps-detailed.md` §6).

## Cross-cutting — the two-flow boundary

Phase 1: the kit flow is **greenfield `INSTALLATION` jobs only**; the native app remains the fallback and stays first-class. Maintenance / replacement / add-product jobs stay on the native flow. Document a clear rule for installers on which app to use for which job.

## Traceability & Diagnostics

End-to-end traceability of the device-operations / pre-paired-kit flow across `safer-ops` and `sensor-alarm-backend`, for diagnostics, debug, and future improvement. When an installer gets stuck or a flow misbehaves, a diagnostician can reconstruct the whole story from one identifier.

### Correlation ID — the universal join key

A `correlationId` (UUID) generated by `safer-ops` per attach attempt is the join key across both systems.

- `safer-ops` generates it when it starts the attach attempt (the `pre-paired-verifications` call) and stores it on the `Kit` (current attempt).
- Sent to the sensor API in the `pre-paired-verifications` **request body** as a new optional `correlationId` field — body, not header (the sensor backend has no middleware to capture/persist a header, and the value must be persisted).
- Persisted by the sensor API on `tbl_device_operations` (new column), copied onto every trail row, and returned in operation responses.
- Carried into the native-app fallback handoff so a kit that falls back to manual stays traceable.

### Sensor backend — transition trail

- New append-only MySQL table **`tbl_device_operation_events`**: `id`, `operationId` (FK → `tbl_device_operations.id`), `correlationId`, `eventType` (`created` | `verify_received` | `alarms_received` | `matched` | `mismatched` | `attaching` | `attached` | `cancelled` | `failed` | `expired`), `fromStatus`, `toStatus`, `detail` JSON, `createdAt`. Indexed on `operationId` and `correlationId`.
- New `correlationId` column on `tbl_device_operations`.
- A **fail-safe** recorder `recordDeviceOperationEvent(...)` in `deviceOperations.entity.ts` — inserts a trail row **and** emits a structured `console.log(JSON.stringify(...))` line. Plain `console.log` is *not* prod-gated (unlike `logToConsole`), so this surfaces in CloudWatch. Wrapped in try/catch: a traceability write must never break the flow it traces (same fail-open principle as Phase 0 fix #1).
- Called at every transition: `createPrePairedVerification`, `handleVerifyResponse`, `handleAlarmsResponse`, `attachPrePairedOperation`, `cancelOperation`, `expireStaleOperations` / `expireIfNeeded`.

### safer-ops — audit trail

- Wire the existing `AuditEvent` Prisma model (currently unwired): a fail-safe `recordAuditEvent(...)` helper writing `prisma.auditEvent`, called for every kit lifecycle + recovery action (create, add device, pair, offsite-test, reserve, attach attempt, verify result, matched / mismatched, repair / swap, retry, native fallback, abort, complete). `entityType="kit"`, `entityId=kitId`; `details` carries `correlationId`, sensor `operationId`, device deltas, actor, result.
- `sensor-client.ts`: log every Sensor API call via the existing Pino logger — method, path, status, duration, `operationId`, `correlationId`, errors.
- A `correlationId` field on the `Kit` (current attempt) for quick lookup.
- Surface for support: `GET /api/kits/:id/audit` returning the kit's `AuditEvent` history + `simulatorTrace`, joinable by `correlationId`.

### The join story

Given one `correlationId`:
- start from `GET /api/kits/:id/audit` → safer-ops `AuditEvent` rows + `simulatorTrace` → the kit-side story including the sensor `operationId`,
- take that `operationId` (or the `correlationId` directly) → query sensor `tbl_device_operation_events` for the transitions,
- the same `correlationId` appears in the sensor backend's prod `console.log` JSON lines (CloudWatch) and in `sensor-client.ts` request/response logs.

### Phasing

The cross-system trail is **Phase 2** (observability hardening) and does not block Phase 1. Existing task **#12** (`Wire AuditEvent for recovery actions`, P1) contributes the kit-side recovery audit; existing task **#20** is extended to surface the kit-audit endpoint described above. The remaining new tasks are tracked as P2 alongside the rest of Phase 2.

## Verification

- **safer-ops, mock mode:** extend `kit-simulator.ts` to emit mismatch outcomes (missing sensor, unexpected sensor, hub no-response); extend `kit-routes.test.ts` to drive the full recovery ladder and assert each transition.
- **Backend interaction test (critical):** confirm that cancelling an in-flight `AWAITING_*` operation stops `subscriber.ts` from intercepting a subsequent native `/users/v1/alarms` install on the same hub.
- **Manual, `prod-controlled-write`:** with a real test kit, force a mismatch (remove a sensor) and walk the recovery ladder; confirm the native-fallback handoff completes the install; confirm an aborted kit releases its reservation.
- Run `scripts/health-check.sh` before/after any change that touches a production path.

## Open items / deferred

- **Partial accept (`attach-with-exceptions`)** — deferred; backend is designed to allow it as a later addition.
- **Auto-converge (automatic re-pairing on-site)** — depends on the firmware confirmation result.
- **Maintenance / replacement kit support** — out of scope for phase 1.
