# MQTT Lab Routing Design

- Date: 2026-05-15
- Scope: design for routing or mirroring selected production MQTT device traffic into QA/UAT/lab workflows without moving the whole field fleet off the production endpoint
- Related systems: production EMQX, `sensor-alarm-backend`, hub firmware, lab devices, QA/UAT backend environments, MQTT topics, device serial registry

## Objective

Define a safe architecture for testing backend changes with a controlled set of physical devices when existing device stock is configured to connect to the production MQTT endpoint.

The intended model is a production-owned lab device list. Devices continue connecting to the production MQTT broker, but only serials on the configured lab list are mirrored or routed to non-production environments. Any command path back to a device must check the same list before publishing.

## Sources Used

- Repository files:
  - `code/sensor-alarm-backend/src/services/mqtt/subscriber.ts`
  - `code/sensor-alarm-backend/src/config/app.ts`
  - `docs/investigations/2026-04-09-mqtt-device-communications.md`
  - `docs/investigations/2026-04-10-prod-user-device-migration-readiness.md`
  - `docs/investigations/2026-04-10-inv-results.md`
  - `docs/investigations/2026-04-12-kafka-message-flow.md`
  - `docs/investigations/2026-05-15-backend-monolith-architecture.md`
- AWS commands or consoles: none
- External references: none

## Findings

The current device topic model already gives a useful routing key:

- hub responses: `sg/sas/resp/<hubSerial>`
- backend commands: `sg/sas/cmd/<hubSerial>`
- backend shared subscription: `$share/sensorGroup/sg/sas/resp/+`

The hub serial number is therefore the natural control point. A lab-routing design should not depend on changing baked firmware endpoints for existing stock. It should classify messages by serial number after they arrive at the production broker.

The current backend design makes direct QA/UAT subscription to production topics unsafe:

- the backend subscriber performs operational side effects, not just passive parsing
- MQTT events can drive MySQL writes, Mongo logs, notification decisions, and follow-up commands
- the shared subscription group means another consumer in the same group could steal messages from production
- a QA backend with publish access to production command topics could send commands to real customer hubs

The safer model is to introduce an explicit routing boundary owned by production.

## Proposed Architecture

Use three separate concepts:

1. **Lab device list**
   - A production-owned registry of hub serials that are allowed to participate in lab routing.
   - The registry can map a serial to one or more targets, such as `qa`, `uat`, `local-dev`, or `shadow-only`.
   - The list should include owner, reason, expiry, and allowed command mode.

2. **MQTT mirror/router**
   - A small production-side service that subscribes to production device response topics.
   - It extracts `<hubSerial>` from `sg/sas/resp/<hubSerial>`.
   - If the serial is on the lab list, it forwards a copy of the message to the configured target environment.
   - If the serial is not on the lab list, it drops the copy and production continues as normal.

3. **Checked command endpoint**
   - A controlled HTTP or internal service endpoint used by QA/UAT to request commands to a physical hub.
   - It checks the serial against the lab list before publishing to `sg/sas/cmd/<hubSerial>`.
   - It records audit information for who/what requested the command, target environment, command body, timestamp, and decision.

This keeps production EMQX as the stable device ingress while avoiding direct write authority from QA/UAT into production MQTT.

## Lab Device List Shape

The lab list should be explicit and time-bound. Suggested fields:

| Field | Purpose |
|---|---|
| `serialNumber` | hub serial extracted from MQTT topic |
| `targetEnvironment` | `qa`, `uat`, `local-dev`, or similar |
| `mode` | `mirror-only`, `command-allowed`, or `disabled` |
| `allowedCommands` | optional command allowlist such as `VERIFY`, `ALARMS`, `BATTERY`, `TEST` |
| `owner` | person or team responsible for the test device |
| `reason` | ticket, test plan, or investigation reference |
| `expiresAt` | automatic safety expiry |
| `createdBy` / `updatedBy` | audit trail |
| `createdAt` / `updatedAt` | audit trail |

For the first implementation, `mirror-only` should be the default. `command-allowed` should require a deliberate change.

## Routing Modes

### Mode 1: Mirror Only

This is the safest first mode.

- Production backend keeps handling device events normally.
- The router copies lab-listed device responses to QA/UAT.
- QA/UAT parses and records what it would have done.
- QA/UAT cannot publish commands back to the device.

Use this to validate parsing, event classification, UI behavior, logging, reporting, and workflow simulation without changing device state.

### Mode 2: Command Gateway

This mode lets QA/UAT send commands only through the checked endpoint.

- QA/UAT calls the command endpoint with serial and command.
- The endpoint verifies the serial is on the lab list.
- The endpoint verifies the command is allowed for that device/mode.
- The endpoint publishes to `sg/sas/cmd/<hubSerial>` only after passing checks.
- Every publish decision is audited.

Use this for controlled physical-device testing such as `VERIFY`, `ALARMS`, `BATTERY`, and `TEST`.

### Mode 3: Routed Lab Workflow

This mode lets a lab-listed device exercise a non-production backend workflow more fully.

- Device still connects to production EMQX.
- Responses are mirrored or routed into a QA/UAT message intake.
- QA/UAT can request allowed commands through the command gateway.
- Production remains the only direct broker-facing authority for non-lab devices.

This should be introduced only after mirror-only mode proves that filtering and audit behavior work.

## Proposed Message Flow

### Device Response Flow

1. Hub publishes to `sg/sas/resp/<hubSerial>` on production EMQX.
2. Production backend continues its existing handling.
3. MQTT mirror/router receives or observes the same message.
4. Router extracts `<hubSerial>`.
5. Router checks the lab list.
6. If no match, router does nothing.
7. If matched, router forwards a copy to the configured target environment.
8. Target environment records the event as mirrored lab traffic.

### Command Flow

1. QA/UAT requests a command through the checked command endpoint.
2. Endpoint validates caller identity and environment.
3. Endpoint checks `<hubSerial>` against the lab list.
4. Endpoint checks command mode and allowed command names.
5. If denied, endpoint returns a denial and writes an audit record.
6. If allowed, endpoint publishes to production EMQX topic `sg/sas/cmd/<hubSerial>`.
7. Endpoint writes an audit record with the publish result.

## Safety Requirements

- QA/UAT backends must not subscribe directly to production shared-subscription groups.
- QA/UAT backends must not have raw production MQTT publish credentials.
- Mirroring must be serial allowlist based, deny by default.
- Command publishing must be deny by default.
- Lab entries must expire automatically unless renewed.
- Commands should be allowlisted by command name, not free-form.
- Production customer devices must remain invisible to QA/UAT unless explicitly listed.
- All routed messages and command attempts must be auditable.
- Payload logs should avoid secrets; serials are operational identifiers and should be treated carefully.

## Implementation Options

### Option A: Application-Level Router Service

Build a small Node service that connects to production EMQX and mirrors only lab-listed serials.

Pros:

- easiest to reason about and test locally
- keeps routing logic in application code
- can use normal DB-backed lab list and audit tables
- can expose the checked command endpoint in the same service

Cons:

- adds another production-side process
- must be carefully configured so it never joins the backend shared subscription group
- needs broker-level subscription permissions

This is the recommended first implementation.

### Option B: EMQX Rule Engine / Bridge

Use EMQX rules or bridges to forward matching messages.

Pros:

- routing is close to the broker
- fewer application moving parts
- may be efficient for high-volume mirroring

Cons:

- dynamic serial allowlists and audit behavior may be harder to manage
- changes live in broker config rather than repo code unless carefully captured
- command authorization still needs an application endpoint

This may be useful later, but it is not the simplest first step.

### Option C: Dedicated Lab Broker With Controlled Bridge

Run a QA/UAT broker and bridge selected serial traffic into it.

Pros:

- clean separation for target environments
- QA/UAT can use a normal MQTT broker locally

Cons:

- more infrastructure
- still requires production-side filtering
- can create confusing ownership if command flow is not centralized

This is useful once mirror-only mode is working and QA/UAT needs a broker-native workflow.

## Recommended First Slice

Implement an application-level router in mirror-only mode.

Minimum first slice:

1. Store a lab list with `serialNumber`, `targetEnvironment`, `mode`, `expiresAt`, `owner`, and `reason`.
2. Router subscribes to production response topics using a distinct client ID and a distinct consumer group name if shared subscriptions are used.
3. Router extracts serials and forwards only matching lab-listed messages.
4. Target environment receives mirrored messages through a separate intake endpoint or queue.
5. Add audit records for forwarded and dropped-by-policy events.
6. Add read-only admin visibility for current lab devices and recent routed messages.

Do not include command publishing in the first slice. Add the checked command endpoint only after mirror-only behavior is proven.

## Command Endpoint Contract

When command mode is added, use a narrow endpoint shape:

```http
POST /internal/mqtt-lab/commands
```

```json
{
  "serialNumber": "HUB_SERIAL",
  "targetEnvironment": "qa",
  "command": "VERIFY",
  "payload": {
    "CMD": "VERIFY"
  },
  "reason": "QA installation-flow test"
}
```

Decision rules:

- reject if serial is not in the lab list
- reject if lab entry is expired
- reject if target environment does not match the lab entry
- reject if mode is not `command-allowed`
- reject if command is not in `allowedCommands`
- reject free-form payloads that do not match known command schemas
- publish only to `sg/sas/cmd/<serialNumber>`

## Open Questions

- Where should the lab list live first: existing production MySQL, a small new service database, or a checked-in config for the very first proof?
- Should target environments receive mirrored messages through HTTP, queue, or a QA MQTT broker?
- Which command names are safe for the first command-enabled lab devices?
- Should a lab-listed device still be processed by production backend, or should any future mode support production suppression for a fully isolated lab device?
- Who owns lab-list approval and expiry renewal?

## Recommended Next Steps

1. Confirm the initial target is mirror-only, not command-enabled.
2. Choose the first 1-3 physical hub serials for the lab list.
3. Decide where the lab list is stored for the first slice.
4. Define the mirrored-message target: QA HTTP intake, queue, or QA broker.
5. Add implementation backlog items for:
   - lab list storage
   - router service
   - target intake
   - audit records
   - later checked command endpoint
