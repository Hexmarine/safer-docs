# Backend Monolith Architecture Review

- Date: 2026-05-15
- Scope: current `sensor-alarm-backend` application shape, monolithic coupling points, and safer modernization boundaries
- Related systems: backend API, MQTT subscriber, Kafka log queue, Socket.IO, Redis, MySQL, MongoDB, SSO, Angular portals, mobile apps, Odoo, PropertyMe, KORE, Twilio, SendGrid/SES

## Objective

Assess whether the Sensor backend is monolithic, describe the current coupling in practical terms, and outline a low-risk modernization path that does not require rewriting production workflows.

## Sources Used

- Repository files:
  - `code/sensor-alarm-backend/src/index.ts`
  - `code/sensor-alarm-backend/src/routes/index.ts`
  - `code/sensor-alarm-backend/src/routes/v1/`
  - `code/sensor-alarm-backend/src/routes/users/v1/`
  - `code/sensor-alarm-backend/src/controllers/`
  - `code/sensor-alarm-backend/src/entities/`
  - `code/sensor-alarm-backend/src/services/`
  - `code/sensor-alarm-backend/src/services/mqtt/subscriber.ts`
  - `code/sensor-alarm-backend/src/services/kafka/producer.ts`
  - `code/sensor-alarm-backend/src/services/kafka/consumer.ts`
  - `code/sensor-alarm-backend/package.json`
  - `docs/investigations/2026-04-09-platform-architecture-and-data-flow.md`
  - `docs/investigations/2026-04-12-kafka-message-flow.md`
- AWS commands or consoles: none
- External references: none

## Findings

The backend is best described as a deployment monolith with domain folders. The source tree has conventional layers such as `routes`, `controllers`, `entities`, `models`, and `services`, but the runtime and data ownership are still tightly unified.

At process startup, `src/index.ts` starts the Express API and, inside the same listening process, requires the MQTT subscriber, Kafka producer, Kafka consumer, socket services, and Socket.IO initialization. That means the API tier is also the device-message worker tier, Kafka consumer tier, and realtime socket tier.

The main backend process owns a wide set of responsibilities:

- admin and user APIs
- property, agency, account, role, lease, job, ticket, invoice, and product workflows
- device and alarm registration flows
- MQTT command publishing and response handling
- alarm event persistence and alert follow-up
- audit logging through Kafka and MongoDB
- direct MySQL operational state updates
- notification delivery paths for email, SMS, and push
- third-party integration logic for Odoo, PropertyMe, KORE, Twilio, SendGrid/SES, S3, Firebase, and related services
- realtime socket handling
- monitoring endpoints

This is not only a large API. It is also the operational workflow engine for the platform.

### Evidence Of Monolithic Coupling

The most important coupling point is runtime coupling. `src/index.ts` loads these services in the same app server process:

- `./services/mqtt/subscriber`
- `./services/kafka/producer`
- `./services/kafka/consumer`
- `./services/socket/main.socket`
- `./services/socket/request.socket`

The second coupling point is workflow size. Several files combine broad business rules, persistence calls, integration calls, and side effects:

| File | Approximate lines | Signal |
|---|---:|---|
| `src/entities/properties.entity.ts` | 31,842 | very broad property/business workflow surface |
| `src/entities/jobs.entity.ts` | 11,721 | large job lifecycle and scheduling surface |
| `src/controllers/users/properties.controller.ts` | 7,415 | large user-facing property API surface |
| `src/entities/endusers.entity.ts` | 6,899 | broad end-user workflow surface |
| `src/services/mqtt/subscriber.ts` | 5,972 | device event handling, persistence, notification, and workflow side effects |
| `src/entities/users.entity.ts` | 5,164 | large user/account workflow surface |
| `src/entities/logs.entity.ts` | 4,116 | audit/event logging fan-out |

The third coupling point is data ownership. The backend imports and writes across both MySQL Sequelize models and MongoDB models from many controllers/entities. The application code, not a separate domain service contract, appears to coordinate state transitions across properties, jobs, alarms, users, logs, subscriptions, and external systems.

The fourth coupling point is asynchronous work placement. Kafka exists, but the prior Kafka investigation found it is mostly an async audit/subscription queue inside the same backend, not a separate event-driven boundary for core device processing. MQTT alarm events are handled directly by `subscriber.ts`; they do not flow through Kafka.

### Current Logical Domains

The current code can be usefully grouped into domains even though it is deployed as one backend:

| Domain | Current code areas | Current boundary quality |
|---|---|---|
| Device and MQTT | `services/mqtt/subscriber.ts`, alarm controllers/entities | high operational importance, low isolation |
| Property and installation | property controllers/entities, job controllers/entities, alarm controllers/entities | core system of record, highly coupled |
| Jobs and scheduling | `jobs.entity.ts`, job controllers, agenda/schedule service references | broad workflow surface, likely coupled to property state |
| Notifications | mail, SMS, push helpers, templates, logs, alarm/job flows | cross-cutting, side-effect heavy |
| Audit and activity logs | Kafka producer/consumer, `logs.entity.ts`, Mongo models | partly asynchronous but still in same deployment |
| Billing/subscriptions/invoices | invoice settings controllers/entities, Odoo references | business workflow coupled to property state |
| Agency/admin/user identity | account/user/role/settings controllers/entities, SSO integration | high-touch product surface |
| Integrations | Odoo, PropertyMe, KORE, Twilio, SendGrid/SES, Firebase, S3 | mixed into domain workflows rather than isolated adapters |
| Monitoring | monitoring routes/controller/entity | reasonably separable surface |

### Why A Rewrite Would Be Risky

A direct rewrite or broad microservice split would be risky because the backend is the source of operational truth for field-device state, property state, job state, notifications, subscriptions, and audit history. Many workflows appear to combine:

1. request validation or MQTT message parsing
2. MySQL writes
3. Mongo log writes
4. notification decisions
5. external API calls
6. follow-up MQTT commands or asynchronous log work

Splitting those without first documenting ownership and contracts would create duplicate-write, missed-alert, inconsistent-property, and failed-installation risks.

## Recommended Modernization Shape

Use a strangler and modularization approach, not a big-bang rewrite.

### Phase 1: Document And Stabilize Boundaries

Create a current-state ownership map before moving code:

- route to controller to entity map for the largest workflows
- MySQL table ownership by domain
- Mongo collection ownership by domain
- MQTT command and response ownership
- Kafka message producer and consumer ownership
- external integration call sites by provider
- notification trigger ownership

Candidate first artifacts:

- backend domain map
- device/MQTT command contract
- property/job state transition map
- notification trigger matrix
- table/collection ownership inventory

### Phase 2: Modularize Inside The Monolith

Before splitting deployments, make boundaries visible inside the existing backend:

- isolate integration adapters behind provider modules
- move domain-specific validation and state transition logic out of mega-files
- add small domain service interfaces around property, job, alarm, notification, and integration operations
- keep persistence writes in one place per domain where possible
- add focused tests around extracted behavior before changing runtime boundaries

Good first candidates:

- notification adapter boundaries
- KORE/SIM adapter boundary
- PropertyMe/Odoo adapter boundaries
- monitoring endpoint logic
- CSV/report/export style flows

Less safe as first candidates:

- MQTT subscriber behavior
- property installation state transitions
- alarm alert persistence and notification suppression logic
- job lifecycle updates that affect property status

### Phase 3: Extract New Workflows Beside The Existing Backend

The safest new service boundary is additive workflow state that can call the existing Sensor API rather than owning existing production tables directly. The prepared-kit / safer-ops style flow fits this model because it can:

- own its own workflow state
- use the existing backend as the system of record for properties, devices, jobs, and alarms
- run locally first
- use mock/simulator validation before any production writes
- avoid competing with the existing backend for MQTT ownership

This creates a clean modernization path without forcing risky extraction from the core backend first.

### Phase 4: Consider Worker Extraction For Natural Async Boundaries

Once behavior is documented and tested, the first separate runtimes to consider are workers, not business systems of record:

- MQTT device-event worker
- notification delivery worker
- scheduled job worker
- audit/log ingestion worker
- integration sync worker for Odoo/PropertyMe/KORE

These are natural candidates because they already behave asynchronously. However, each needs idempotency, ownership, and retry semantics defined before extraction.

## Candidate Extraction Matrix

| Candidate | Value | Risk | Suggested timing |
|---|---|---|---|
| Monitoring endpoints | isolates health/ops checks | low | early |
| Notification delivery worker | reduces side effects inside core flows | medium | after trigger matrix |
| Integration adapters | reduces provider-specific coupling | medium | early to middle |
| Audit/log worker | Kafka already provides a partial boundary | medium | after replay/idempotency review |
| Scheduled job worker | separates background work from API latency | medium-high | after job state map |
| New safer-ops/prepared-kit service | clean additive product boundary | medium | early, beside monolith |
| MQTT worker | isolates critical device path | high | later, after command contract and tests |
| Property/job core service | clearer ownership long term | very high | last, only after table/state ownership is proven |

## Risks Or Constraints

- Device/MQTT behavior is production-critical; extraction mistakes could affect installs, alarm event handling, SMS/push/email notifications, or hub state reconciliation.
- The same API process currently subscribes to MQTT and runs Kafka consumers; operational scaling changes may affect side-effect processing, not just HTTP throughput.
- Shared MySQL and Mongo access from many modules means code boundaries are looser than folder names imply.
- Kafka is not the core device-event backbone today, so introducing Kafka into the MQTT path would be an architecture change, not a simple extraction.
- Some workflows likely depend on implicit ordering between DB writes, logs, notifications, and external calls.
- The codebase has tests, but extraction should start by measuring coverage around the exact workflow being moved.

## Cost Optimization Opportunities

This review was not a cost investigation. Potential indirect cost benefits of modularization include:

- smaller worker runtimes for background tasks
- independent scaling of API and async processing
- clearer shutdown/startup behavior in non-production environments
- reduced operational debugging time when a background side effect fails

These should be treated as secondary benefits. Reliability and change safety are the primary drivers.

## Recommended Next Steps

1. Produce a backend domain map from `routes`, `controllers`, `entities`, `models`, and `services`.
2. Start with one workflow trace: property installation from API request to MQTT command to DB/log/notification side effects.
3. Build a table/collection ownership inventory for the installation and alarm-event domains.
4. Define a notification trigger matrix before changing notification code paths.
5. Keep new safer-ops/prepared-kit workflow state outside the legacy backend and integrate through explicit API calls.
6. Avoid extracting MQTT handling until command contracts, idempotency rules, and tests are clear.

## Open Questions

- Which backend workflows are currently the most painful to change: installation, notifications, invoices, admin/agency management, or integrations?
- Are duplicate MQTT subscriptions across API instances intentional and fully safe for every command/event type?
- Which MySQL tables are the true system of record for hub, alarm, property, and job lifecycle state?
- Which Mongo collections are audit-only versus operationally queried by product screens?
- Is the desired future shape one modular backend plus workers, or multiple product services around the existing backend API?
