# Production User and Device Migration Readiness

- Date: 2026-04-10
- Scope: confirm that real production users and live communicating devices exist in the current account (`747293622182`), and identify what must migrate to the new JV AWS account to preserve that state
- Related systems: EMQX MQTT broker, sensor-prod RDS MySQL, tbl_admins, tbl_users, tbl_alarms, tbl_properties, SmokeAPI-LB, Kafka, KORE SIM management

## Objective

Determine whether the current production platform has live tenants/customers and active field devices communicating, as the primary evidence for treating the account-to-account migration as business-critical rather than theoretical. This note does not repeat the full infrastructure migration matrix (see [2026-04-09-prod-external-account-migration-evaluation.md](2026-04-09-prod-external-account-migration-evaluation.md)); it focuses on the live-user and live-device evidence.

## Sources Used

- Repository files:
  - `code/sensor-alarm-backend/src/models/mysql/users.ts`
  - `code/sensor-alarm-backend/src/models/mysql/admins.ts`
  - `code/sensor-alarm-backend/src/models/mysql/alarms.ts`
  - `code/sensor-alarm-backend/src/models/mysql/properties.ts`
  - `code/sensor-alarm-backend/src/constants/db_models.ts`
  - `code/sensor-alarm-backend/src/constants/app.ts`
  - `docs/investigations/2026-04-09-mqtt-device-communications.md`
  - `docs/investigations/2026-04-09-prod-external-account-migration-evaluation.md`
- AWS commands:
  - `aws ec2 describe-instances --filters vpc-id=vpc-03ec09702866ca7b1 instance-state-name=running`
  - `aws cloudwatch get-metric-statistics` for EMQX prod `NetworkIn` and `NetworkOut` (48 h, hourly)
  - `aws cloudwatch get-metric-statistics` for Kafka prod `NetworkIn` (24 h, hourly)
  - `aws cloudwatch get-metric-statistics` for `sensor-prod` RDS `DatabaseConnections` (48 h, hourly)
  - `aws cloudwatch get-metric-statistics` for `SmokeAPI-LB` and `sso-api-lb` `RequestCount` (12 h, hourly)
  - `aws rds describe-db-instances --region ap-southeast-2`
  - `aws elbv2 describe-load-balancers --region ap-southeast-2`

## Findings

### 1. Platform User Model

The application stores users across two separate MySQL tables in `sensor-prod`:

**`tbl_admins`** — all business-facing and operational users. A single table handles multiple persona types via a `userType` integer discriminator:

| userType | Code | Role |
|---|---|---|
| 1 | `superadmin` | Super Admin (platform staff) |
| 2 | `agency` | Agency (property management company) |
| 3 | `agent` | Agent (individual within an agency) |
| 4 | `tradeperson` / `contractor` | Trade Person / Contractor (installs alarms) |
| 5 | `servicestaff` | Service Staff |
| 6 | `tenant` | Tenant |
| 7 | `landlord` | Asset Owner / Landlord |
| 8 | `subadmin` | Sub Admin |

Each admin record carries: `email`, `phone`, `name`, `companyName`, `status`, `roleId`, `permission`, `agency` (parent agency FK), `noOfProperties`, `noOfJobs`, business identifiers, integration keys (PropertyMe tokens, Odoo keys, KORE keys), and notification preferences. Status codes are `0=Inactive, 1=Active, 2=Deleted, 3=Blocked`.

**`tbl_users`** — mobile app end users. Stores `uniqueID` (device identifier), `deviceToken` (push notification token), `deviceType` (`android`/`ios`), and `notificationSetting`. These represent property tenants or owners who have installed the SensorGlobal mobile app.

The platform is a multi-tenant hierarchy: Agency → Agents and Trade Persons → Properties → Alarm Hubs and Devices.

**`tbl_properties`** — physical addresses. Each property links to an agency, an agent, and a landlord. Properties have their own status, timezone, geographic coordinates, alarm test schedules, and lease tracking.

### 2. Device (Alarm and Hub) Model

All field devices — both hub controllers and individual smoke alarms — are stored in **`tbl_alarms`**. The table uses self-referential foreign keys to model the hub-spoke relationship:

- A hub/controller record has `controller = 0` and `controllerId = NULL`.
- A child alarm record has `controllerId` pointing to its parent hub's `id`.

**Key status fields:**

| Field | Values | Meaning |
|---|---|---|
| `connectedStatus` | `0=Not Connected, 1=Connected, 2=Removed, 3=In Progress, 4=Failed` | Current MQTT connectivity state |
| `status` (ALARM_STATUS) | `active/inactive/...` | Record-level lifecycle status |
| `simId` | string | KORE SIM card ICCID |
| `simStatus` | `inactive/active/ready/scheduled` | SIM card status per KORE |
| `batteryStatus` | string | Last reported battery state |
| `powerState` | tinyint | Mains power state |
| `alertStatus` | tinyint | Active alert state |
| `tamperedStatus` | tinyint | Tamper event state |
| `propertyId` | FK | Which property this device is installed at |
| `serialNumber` | string | Physical device serial |

A device with `connectedStatus = 1` (CONNECTED) is actively maintaining an MQTT session with the broker. Any device that has sent a message recently will have an updated `updatedAt` timestamp and a battery or power state populated.

The KORE SIM integration means every hub has a cellular identity registered in a third-party SIM management platform in addition to the platform database. Both must be migrated in sync.

### 3. Live MQTT Traffic — Confirmed Active

The production EMQX broker (`Emqx-prod`, instance `i-0b381a2bfdf65a329`, `c5.xlarge`, public IP `13.211.102.112 = mqtt.sensorglobal.com`) showed continuous network traffic over the observed 48-hour window (Apr 9–10 2026):

**NetworkIn (data received by broker from field devices):**

| Window | Baseline (typical) | Peak |
|---|---|---|
| Apr 9 00:00–23:59 UTC | ~750 KB–1 MB per hour, every hour | 16.5 MB at 05:00 UTC |
| Apr 10 00:00–19:59 UTC | ~750 KB–1 MB per hour, every hour | 19.9 MB at 08:00 UTC |

**NetworkOut (data sent by broker to field devices):**

| Window | Typical range |
|---|---|
| Apr 9–10 UTC | ~620–700 KB per hour, consistent across all 48 hours |

**Interpretation:**

- The flat ~750 KB–1 MB per hour baseline on NetworkIn is characteristic of MQTT keep-alive pings from a stable population of persistently connected hubs. MQTT keep-alives are short, high-frequency, and produce a relatively smooth traffic curve.
- The occasional large spikes (16–20 MB) are consistent with batch reconnection events, firmware check responses, or bulk `ALARMS`/`BATTERY` inventory commands being issued to multiple hubs simultaneously.
- There was **no extended quiet period** in 48 hours, which means devices were connected and communicating continuously around the clock.
- NetworkOut being slightly lower than NetworkIn is consistent with the backend sending brief command ACKs and periodic command sequences, while devices are always sending keep-alives plus event payloads.

**EMQX management API:** The CWAgent metrics for `Emqx-prod` report only `disk_used_percent` and `mem_used_percent`. There are no custom EMQX metrics being pushed to CloudWatch (e.g., no `emqx.connections.count` metric was found). The EMQX HTTP management API (default port 18083) is not referenced in the application codebase. Session and connection counts are not directly observable from AWS metrics without direct broker access.

### 4. Kafka Pipeline — Confirmed Active

The production Kafka host (`prod-kafka`, instance `i-09d176fd791fe7c64`, `t3.medium`, `13.237.253.114 = kafka.sensorglobal.com`) showed:

- NetworkIn: ~1.6–1.7 MB per hour baseline throughout Apr 10, with spikes to ~17.5 MB (00:00 UTC) and ~21.3 MB (10:00 UTC).
- This confirms the Kafka event ingestion pipeline is actively receiving records, consistent with device events and application-generated messages being produced continuously.

### 5. Production API Traffic — Confirmed Active

The `SmokeAPI-LB` (production API load balancer, 5 × `prod-api-server` t3.medium targets behind it) handled:

| Hour (UTC) | Requests |
|---|---|
| 08:00–09:00 | 8,487 |
| 09:00–10:00 | 8,502 |
| 10:00–11:00 | 8,438 |
| 11:00–12:00 | 8,418 |
| 12:00–13:00 | 8,587 |
| 13:00–14:00 | 8,446 |
| 14:00–15:00 | 8,321 |
| 15:00–16:00 | 8,237 |
| 16:00–17:00 | 8,381 |

Approximately **8,300–8,600 API requests per hour** (~140 per minute, ~2.3 per second) sustained throughout the Apr 10 observation window. This traffic level is consistent with:
- Active users accessing the Angular portals (`admin`, `agent`, `trader`, `owner`)
- Background polling from the mobile apps
- Periodic device-driven API calls (job status checks, notification delivery, property state sync)

The SSO ALB (`sso-api-lb`) handled 91–401 authentication requests per hour, confirming that users are actively logging in and maintaining sessions.

### 6. Production Database — Active and Sized for Production Load

The `sensor-prod` RDS instance:

| Property | Value |
|---|---|
| Engine | MySQL |
| Instance class | `db.t3.medium` |
| Multi-AZ | Yes |
| Allocated storage | 20 GB |
| Free storage | ~8.86 GB (as of Apr 10) |
| Used storage | ~11.1 GB (55% of allocated) |
| Status | `available` |
| Average DB connections | 4–5 per hourly average |

The 55% storage utilization (11.1 GB in use) across all production application tables confirms a substantial active database. Given the schema includes alarm logs, event history, job records, property files, subscriptions, and full user records with media and documents, this volume is consistent with months or years of real operational data.

The average connection count of 4–5 per hour appears low but reflects hourly averages; connection pooling across 5 `t3.medium` API servers means instantaneous connections will be higher. The `sensor-prod` instance is Multi-AZ, confirming it is production-classified by the prior platform team.

The `odoo-production` RDS instance (`db.t3.medium`, PostgreSQL, 100 GB) is also active. Odoo holds CRM, billing, and business operational data for the agency and contractor workflows.

### 7. Production Infrastructure Summary

The following infrastructure is live and active in the prod VPC (`vpc-03ec09702866ca7b1`) as of Apr 10 2026:

| Instance | Name | Type | Public IP | Role |
|---|---|---|---|---|
| i-0b381a2bfdf65a329 | Emqx-prod | c5.xlarge | 13.211.102.112 | MQTT broker (= mqtt.sensorglobal.com) |
| i-09d176fd791fe7c64 | prod-kafka | t3.medium | 13.237.253.114 | Kafka broker (= kafka.sensorglobal.com) |
| i-0b53bd5e7a9b9dec0 | prod-api-server | t3.medium | — | API server (1 of 5) |
| i-09acc276e638ba099 | prod-api-server | t3.medium | — | API server (2 of 5) |
| i-0bcc2dcab7c6dfedf | prod-api-server | t3.medium | — | API server (3 of 5) |
| i-0277626c3f4828de7 | prod-api-server | t3.medium | — | API server (4 of 5) |
| i-035d91e3857aaea0e | prod-api-server | t3.medium | — | API server (5 of 5) |
| i-0d120e1ad6a527ea0 | prod-sso-api | t3.small | — | SSO / OIDC service |
| i-07b498016eab37ae1 | Smoke-prod-Keychain-app | t3.small | — | Keychain service |
| i-0c5073e95aba77614 | Odoo-production | t3.large | 54.253.158.197 | Odoo CRM / ERP (= sensorglobal.com) |
| i-0a1cd47cfadedf035 | smoke-prod-api-cron-server | t2.micro | 54.153.177.72 | Background cron jobs |
| i-0bd9a5e391e812fcc | wordpress-prod | t3.medium | — | WordPress portal |
| i-0dc7e540dc021d6e9 | wordpress-sensor-insure | t2.medium | 54.252.123.97 | SensorInsure WordPress |
| i-0908eff051972cd8c | Power-BI-Gateway | t3.large | — | Power BI data gateway |
| i-00abd001f35108630 | openvpn-prod | t2.micro | 13.211.118.57 | VPN access |

Active ALBs serving production traffic:
- `SmokeAPI-LB` → 5× prod-api-server (8,300–8,600 req/hr)
- `sso-api-lb` → prod-sso-api (91–401 req/hr)
- `keychain-production-ALB` → Smoke-prod-Keychain-app
- `wordpress-prod-alb` → wordpress-prod

## What Must Migrate

Based on the user model, device model, and observed live activity, the following must be migrated to the new JV account with zero data loss to preserve the production user-device relationships:

### Critical data (cannot be rebuilt)

| Asset | Location | Migration concern |
|---|---|---|
| Production application database | `sensor-prod` MySQL RDS (~11 GB) | All `tbl_admins` (platform users), `tbl_users` (mobile users), `tbl_alarms` (hub and device records with connectedStatus, simId, serialNumber), `tbl_properties`, all related tables | Must be migrated with point-in-time consistency; snapshot+restore or replication |
| MongoDB alarm and event logs | MongoDB Atlas `sensor-prod.vr48f.mongodb.net` | `tbl_alarm_logs` (MySQL) references MongoDB ObjectIDs; Mongo holds raw event payloads | Must migrate or re-point Atlas together with MySQL to avoid broken references; see `2026-04-10-inv-results.md` |
| Odoo production database | `odoo-production` PostgreSQL RDS (100 GB) | CRM, billing, subscription, and business operational data tied to real users | Snapshot+restore or dump/restore |
| KORE SIM mappings | `tbl_alarms.simId` → KORE SIM management API | Each hub has a SIM registered in KORE; the simId/simStatus fields must match what KORE reports | KORE API keys must be re-registered with the new account identity |
| Redis transient state | Production ElastiCache | Short-lived dedupe/cache keys with TTLs; user sessions are stored in MySQL `tbl_sessions` | Can be cold-started safely in a maintenance window; see `2026-04-10-inv-results.md` |

### MQTT broker migration (highest sensitivity)

The MQTT broker is the most migration-sensitive component:

- All currently connected hubs have an open MQTT session to `13.211.102.112` (the prod EMQX host).
- The hubs connect by IP-backed DNS (`mqtt.sensorglobal.com = 13.211.102.112`). If the new account's MQTT broker is reachable at a new IP and the DNS record is changed, hubs will attempt reconnection automatically on their next keep-alive timeout or connection drop.
- The MQTT topic structure is `sg/sas/cmd/<serial>` and `sg/sas/resp/<serial>`. The serial number is the hub's identity in the broker, and that identity is already in `tbl_alarms.serialNumber`.
- **Risk**: if broker credentials change (username/password) during migration, existing hubs cannot reconnect until they receive updated credentials. The code shows broker credentials are environment variables on the backend; hub credentials are not visible in the codebase and may be static.
- **Risk**: broker ACL configuration (if any) exists only on the EMQX host, not in this repository. It must be exported from the current broker and imported to the new one before cutover.
- **Risk**: the EMQX instance runs on `c5.xlarge`, which is relatively large. Its current in-memory MQTT session state (connected clients, QoS 0/1 queued messages) is not migratable and will reset on broker replacement. All hubs will briefly disconnect and reconnect.

### User and portal continuity

- Platform business users (`tbl_admins`) access the platform via the Angular portals at `admin.sensorglobal.com`, `agent.sensorglobal.com`, `trader.sensorglobal.com`, `owner.sensorglobal.com`.
- Session tokens are stored in MySQL `tbl_sessions`, not Redis. A Redis cold-start should not invalidate sessions by itself, though users may still need to re-authenticate if auth secrets, cookies, domains, or deployment timing change during cutover.
- Mobile app users (`tbl_users`) use push notification device tokens. The token values will survive migration but the push notification backend (APNs/FCM via SendGrid/Twilio) must be reconfigured on the new account.
- PropertyMe API tokens (`tbl_propertyme_api_tokens`), KORE keys (`tbl_kore_api_keys`), and Odoo API keys (`tbl_odoo_api_keys`) are account-level integration credentials that will need to be re-validated or re-issued by those third parties for the new account identity.

## Risks and Constraints

1. **Zero tolerance for MQTT broker downtime during business hours.** The MQTT traffic is continuous 24/7. Any window where the broker is unreachable will cause hubs to queue events locally (if firmware supports it) or drop events entirely. The migration must include a warm-standby broker or DNS-based cutover with minimal reconnection gap.

2. **Hub credentials are broker-managed and shared.** Later EMQX host investigation confirmed one shared broker credential in the built-in auth database, used by both API servers and field devices. Preserve the credential model for migration, but treat shared credentials as a security risk and rotate only through a planned device-safe process.

3. **SIM management is a third-party dependency.** KORE SIM records (simId, simStatus) are linked to the current account. KORE API key migration must be coordinated with KORE, and the new account must be able to query and update SIM status before the first device reports a SIM event.

4. **11 GB of production MySQL data with active writes.** The database receives writes continuously (alarm events, job updates, notification logs). A simple snapshot-and-restore migration will have a data gap proportional to the cutover window. A replication-based approach (read replica pointing to new account) would minimize data loss but adds operational complexity.

5. **Active sessions may be disrupted by auth/domain cutover, not by Redis alone.** Redis can be cold-started, but any change to session signing, cookies, SSO, hostnames, or database consistency can still force re-authentication. Schedule cutover in an off-peak Australian window and communicate expected login disruption.

6. **Odoo 100 GB database.** The `odoo-production` database is 100 GB and uses `db.t3.medium` with no Multi-AZ. It is larger than the main application database and will require its own migration plan. Odoo has application-level state (running background workers, cron jobs) that must also be stopped and restarted cleanly.

7. **Direct IP endpoints.** Both `mqtt.sensorglobal.com` (13.211.102.112) and `kafka.sensorglobal.com` (13.237.253.114) are direct A records pointing to public IPs of specific EC2 instances. New instances in the new account will have different IPs. DNS record changes require TTL-aware planning; if device firmware caches the resolved IP, a DNS change alone may not be sufficient.

## Recommended Next Steps

- **Preserve the confirmed MQTT credential model**: production hubs and API servers share one EMQX broker credential. The new broker must be pre-loaded with that model before DNS cutover; any rotation should be a separate device-safe security project.
- **Export current MQTT broker ACL and authentication config**: before any migration work begins, take a snapshot of the EMQX configuration from the prod host so it can be reproduced exactly in the new account.
- **Plan MySQL migration approach**: decide between:
  - snapshot + restore (simple, but has downtime gap)
  - read replica + promote (near-zero data loss, more complex)
  - mysqldump + replay (suitable for small tables)
  Given 11 GB and continuous writes, a read replica approach is recommended.
- **Plan MQTT broker cutover sequence**: decide between:
  - parallel broker running in new account with DNS flipped (preferred)
  - blue/green broker with short DNS TTL and forced reconnect window
- **Engage KORE for SIM management migration**: the new account will need its own KORE API key pair. Each hub's simId must remain registered and active in KORE under the new key context.
- **Schedule Odoo migration separately**: plan a dedicated investigation for Odoo migration, including background job quiescing, data export, and post-migration validation of subscription and billing records.
- **Establish a production user headcount**: run a read-only query against `sensor-prod` to count active admin records by userType before migration planning begins. This establishes the migration scope in concrete terms.

## Open Questions

- What firmware version and MQTT client library do current production hubs use? Does the firmware retry on connection failure, or does it require a manual reconnect?
- Are hub MQTT credentials rotatable remotely (via an over-the-air config update), or are they burned in at provisioning time?
- Is there a log or metric showing the number of currently connected MQTT clients? (EMQX management API on port 18083, if accessible, could answer this.)
- Does the new JV AWS account already exist? If so, what region is targeted, and is it also `ap-southeast-2`?
- What is the decommission deadline for the current account? A hard deadline changes the migration sequencing significantly.
- Are any third-party integrations (PropertyMe, Odoo SaaS, KORE) currently bound to the `747293622182` account identity in a way that requires a formal re-onboarding process (signed agreements, IP allow-lists, account approval)?
