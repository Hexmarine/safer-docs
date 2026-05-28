# Platform Architecture And Data Flow

- Date: 2026-04-09
- Scope: infer the current end-to-end Sensor platform architecture and the main device-to-alarm operational flow from upstream docs, application code, and prior AWS inventory work
- Related systems: Sensor Hub, smoke alarms and other field devices, MQTT, backend API, SSO, Angular portals, mobile apps, RDS, MongoDB, Kafka, Odoo, PropertyMe, notification services, AWS ingress and hosting

## Objective

Describe how the platform appears to work overall, with extra focus on the path from an alarm-capable device event through the hub, cloud backend, operational workflows, and user-facing portals.

## Sources Used

- Repository files:
  - `docs/upstream/Sensor Technical Brief.docx`
  - `docs/upstream/Sensor Global System Description - Rev 6 - July 28 2025.docx`
  - `docs/upstream/SENSOR ARCHITECHTURE DOCUMENT_JAN2025.docx`
  - `docs/investigations/2026-04-09-ap-southeast-2-architecture-baseline.md`
  - `docs/investigations/2026-04-09-code-component-review.md`
  - `code/sensor-alarm-backend/src/index.ts`
  - `code/sensor-alarm-backend/src/utils/BootStrap.ts`
  - `code/sensor-alarm-backend/src/services/mqtt/subscriber.ts`
  - `code/sensor-alarm-backend/src/controllers/alarms.controller.ts`
  - `code/sensor-alarm-backend/interfaces/config/app.interface.ts`
  - `code/sensor-alarm-backend/appspec.yml`
  - `code/sensor-angular/src/environments/set-prod-env.ts`
  - `code/sso-provider/appspec.yml`
- AWS commands or consoles:
  - prior inventory work recorded in `docs/investigations/2026-04-09-ap-southeast-2-architecture-baseline.md`
  - prior Route 53, ALB target group, and S3 bucket inspection performed during this investigation session
- External references: none

## Findings

- The platform is best understood as a multi-environment IoT operations platform rather than a simple alarm backend.
- The likely production path is:
  1. field alarm or sensor event
  2. Sensor Hub over local RF
  3. hub backhaul over LTE Cat-M using KORE SIM
  4. MQTT broker tier
  5. Node/Express backend API on EC2
  6. stateful updates in MySQL plus event/audit logging in MongoDB
  7. downstream notifications, jobs, and business workflows
  8. presentation in Angular portals, mobile apps, and internal Odoo workflows
- Upstream docs say the hub uses `433Mhz FSK RF` to talk to devices and `LTE Cat-M` to talk back to the platform, with the hub acting as the bridge in a `hub + spoke topology`.
- Current AWS evidence and code both align with a split hosting model:
  - Angular portals are delivered through CloudFront-backed public domains, with matching S3 buckets for web artifacts
  - API and SSO are served via internet-facing ALBs to EC2 instance target groups
  - MQTT and Kafka also appear as separately addressed public endpoints
- The main runtime appears to be monolithic or near-monolithic in the backend:
  - one Node/Express API handling device management, jobs, notifications, reporting, and major integrations
  - one separate SSO/OIDC service
  - Odoo as the internal CRM / operations system

### High-level architecture

- Device and edge layer:
  - Sensor smoke alarms and other supported devices communicate with a local Sensor Hub
  - the hub is the network boundary for the premises and forwards only operational status signals such as connectivity, alerting, tamper, and battery
- Messaging and control layer:
  - MQTT is the primary command and telemetry path between backend and hub
  - the backend subscribes to shared MQTT response topics and publishes commands back to per-hub command topics
- Core application layer:
  - the main backend is a Node/Express application with Swagger, sessions, Socket.IO-style realtime wiring, Sentry, and New Relic instrumentation
  - the app boots both MySQL and Mongo connections on startup
- Data layer:
  - MySQL in RDS appears to hold operational state, users, properties, alarms, jobs, and broader business records
  - MongoDB stores high-volume event and alarm logs
  - Kafka is present in code and AWS footprint and is described upstream as part of high-performance event logging
  - Redis is also referenced in code for supporting state and message-adjacent workflows
- Experience and operations layer:
  - Angular portals exist for `admin`, `agent`, `trader`, `owner`, `activation`, and `reschedule`
  - native iOS and Android apps also participate
  - Odoo is used for CRM and internal operations
  - PropertyMe is actively integrated for property-management workflows
  - Twilio/SendGrid support outbound notifications

### What happens when the backend starts

- `code/sensor-alarm-backend/src/index.ts` shows the app starts Express, loads middleware, initializes Sentry and Swagger, mounts routes/controllers, and starts the API server.
- `code/sensor-alarm-backend/src/utils/BootStrap.ts` shows startup connects to:
  - MySQL through Sequelize
  - MongoDB archive connection
  - MongoDB primary connection
- `code/sensor-alarm-backend/interfaces/config/app.interface.ts` confirms the expected external dependencies:
  - AWS and S3
  - SendGrid
  - Twilio
  - MQTT
  - Kafka
  - host/domain settings for the web estates

### Device and alarm event flow

- The clearest operational flow is in `code/sensor-alarm-backend/src/services/mqtt/subscriber.ts`.
- On connect, the backend subscribes to:
  - `$share/sensorGroup/sg/sas/resp/+`
- The trailing topic element is treated as the controller or hub serial number.
- Incoming MQTT payloads are parsed as command-style events, including:
  - `VERIFY`
  - `ADD`
  - `REMOVE`
  - `TEST`
  - `ALERT`
  - `TAMPERED`
  - `DISCONNECT`
  - `RECONNECT`
  - `BATTERY`
  - `ALARMS`
  - `POWER`
  - `HUSH`
  - `RESET`
- The backend then uses those events to update persistent state and trigger follow-on work. Observed behaviors include:
  - mark hub or alarm connected, verified, tested, disconnected, or reconnected
  - update battery and mains power state
  - sync the set of alarms associated with a controller
  - record event history in Mongo-backed alarm logs
  - send notifications and push notifications
  - schedule or suppress alert-related follow-up actions
  - refresh SIM-related status checks
- The backend also drives the hub from the server side by publishing commands to:
  - `sg/sas/cmd/<controllerSerialNumber>`
- Observed command chaining strongly suggests an inventory/synchronization loop:
  - after `VERIFY`, request `ALARMS`
  - after `ALARMS`, request `BATTERY`
- This implies the cloud does not just receive alarms. It actively polls or orchestrates hub state to keep the platform inventory aligned with what the hub currently sees.

### Alarm management from the application side

- `code/sensor-alarm-backend/src/controllers/alarms.controller.ts` shows application users can trigger hub actions via API endpoints.
- Examples:
  - removing an alarm sends `{"CMD":"REMOVE","ALARMSERIAL":"..."}`
  - changing an alarm sends a `REMOVE` followed by an `ADD`
  - other flows call `VERIFY` and related controller commands
- This means operator actions in the web or mobile apps are not just database updates. They can result in immediate MQTT commands to the field hub.

### User-facing and business-facing systems

- `code/sensor-angular/src/environments/set-prod-env.ts` shows the main production web surfaces:
  - `https://api.sensorglobal.com/api/v1/`
  - `https://admin.sensorglobal.com/`
  - `https://agent.sensorglobal.com/`
  - `https://trader.sensorglobal.com/`
  - `https://activation.sensorglobal.com/`
  - `https://reschedule.sensorglobal.com/`
  - `https://owner.sensorglobal.com/`
- The same config also shows:
  - S3-hosted asset bucket usage
  - deep links into mobile apps
  - PropertyMe authorization via the main API
- Backend searches show broad business workflow scope beyond raw alarm handling:
  - property/job lifecycle logic
  - PropertyMe synchronization and document posting
  - Odoo updates for property and subscription status
  - audit/report generation from Mongo alarm logs

### AWS runtime placement

- Prior AWS architecture work already showed:
  - separate `prod`, `qa`, `dev`, `sandbox`, and `uat` VPCs in `ap-southeast-2`
  - EC2-backed API and SSO tiers behind internet-facing ALBs
  - RDS MySQL for the main platform and PostgreSQL for Odoo
- This session’s AWS inspection adds strong public entry-point evidence:
  - Route 53 records point `api.*` hostnames to ALBs
  - Route 53 records point `auth.*` hostnames to SSO ALBs
  - portal hostnames such as `admin.sensorglobal.com` and `agent.sensorglobal.com` point to CloudFront distributions
  - S3 bucket names line up with those portal and API environments
  - MQTT and Kafka hostnames resolve directly to environment-specific public IPs
- Combined with `appspec.yml` files in the API and SSO repos, the deployment pattern looks like EC2 plus CodeDeploy-style hooks plus process supervision rather than containers or ECS.

### Best current model of the alarm path

1. A smoke alarm or related device changes state, such as alert, tamper, low battery, test, or connectivity.
2. The Sensor Hub receives that event over local RF.
3. The hub sends a command-style MQTT response upstream over its cellular backhaul.
4. The backend receives the event on a shared MQTT subscription keyed by hub serial number.
5. The backend updates operational state in MySQL and writes event/audit records into Mongo.
6. The backend may immediately issue follow-up MQTT commands back to the hub to verify state, refresh alarm inventory, or query battery condition.
7. The backend emits notifications, updates jobs/workflows, and exposes the changed state to web/mobile users and to internal business systems.
8. Business-side integrations then consume or reflect the operational outcome:
  - property-management workflows through PropertyMe
  - CRM/billing or internal operations through Odoo
  - user communications through email/SMS/push

## Risks Or Constraints

- This is an inference-driven architecture summary, not a packet capture or production trace.
- The upstream docs mention `distributed HiveMQTT`, while the current AWS footprint shows EC2 hosts named like `Emqx-*`; the exact broker product in current production is not yet fully reconciled.
- Kafka is clearly present in both docs and code, but this pass did not isolate its exact real-time role versus logging/analytics role.
- Frontend hosting is strongly consistent with `S3 + CloudFront`, but this pass did not inspect CloudFront distributions directly.
- The local `firmware` clone is incomplete, so the hub/device-side software could not be reviewed directly.

## Cost Optimization Opportunities

- The architecture duplicates most major components across `prod`, `qa`, `dev`, `sandbox`, and `uat`, which likely increases always-on cost for ALBs, NAT gateways, RDS, caches, VPN, broker hosts, and support systems.
- MQTT broker, Kafka, Odoo, and non-prod API estates are likely good candidates for environment rationalization if some environments are lightly used.
- The device flow is notification-heavy and audit-heavy, so any cost reduction around metric streams, logging, and long-retention event stores should be reviewed carefully against SOC/compliance needs.

## Recommended Next Steps

- Trace one or two alarm event types more deeply through:
  - `alarmEntity`
  - `jobsEntity`
  - notification helpers
  - Kafka producer/consumer code
- Inspect frontend role separation to map which personas use `admin`, `agent`, `trader`, and `owner` in day-to-day workflows.
- Validate the current MQTT broker stack directly from infrastructure naming and configs so the `HiveMQTT` versus `EMQX` question is settled.
- Inspect CloudFront distributions and TLS/certificate bindings to complete the public ingress map for the Angular portals.
- Repair or re-clone `firmware` if device-side behavior needs to be confirmed rather than inferred from cloud code.

## Open Questions

- Which device event types are considered compliance-significant versus only operationally informative?
- What exact role does Kafka play today: real-time fan-out, log buffering, analytics ingestion, or all three?
- Is the current production MQTT tier HiveMQ, EMQX, a migration between them, or a mixed topology?
- How much of the property and jobs workflow is driven directly from alarm events versus user-initiated scheduling and support processes?
