# RDS and ElastiCache Cost Optimisation

- Date: 2026-05-06
- Scope: Production RDS and ElastiCache resources in `ap-southeast-2`
- Related systems: `odoo-production`, `sensor-prod`, `smokeprodapiredis`, Terraform prod stack

## Objective

Identify safe cost optimisation candidates for production RDS and cache resources using read-only AWS evidence and the checked-in Terraform prod configuration.

## Sources Used

- Repository files:
  - `code/infra/terraform/environments/prod/generated_data.tf`
  - `code/infra/terraform/environments/prod/variables.tf`
  - `code/infra/terraform/environments/prod/optim.tfvars`
  - `docs/investigations/2026-04-11-rds-deep-analysis.md`
  - `docs/low-traffic-optimisation-backlog.md`
- AWS commands:
  - `aws sts get-caller-identity`
  - `aws rds describe-db-instances`
  - `aws rds describe-reserved-db-instances`
  - `aws elasticache describe-replication-groups`
  - `aws elasticache describe-cache-clusters`
  - `aws elasticache describe-reserved-cache-nodes`
  - `aws cloudwatch get-metric-statistics`
  - `aws ce get-cost-and-usage`
  - `aws pricing get-products`
- External references: none

## Findings

### Current Production RDS Shape

| DB instance | Engine | Class | Storage | Multi-AZ | Status |
|---|---:|---:|---:|---:|---:|
| `odoo-production` | PostgreSQL 14.17 | `db.t3.medium` | 100 GiB `io1`, 3,000 IOPS | false | available |
| `sensor-prod` | MySQL 8.0.44 | `db.t3.medium` | 20 GiB `gp2` | true | available |

Latest 30-day CloudWatch sample, `2026-04-06` to `2026-05-06`:

| DB instance | Read IOPS avg / max | Write IOPS avg / max | CPU avg / max | Connections avg / max |
|---|---:|---:|---:|---:|
| `odoo-production` | 0.79 / 742.08 | 5.08 / 291.56 | 6.89% / 57.26% | 19.76 / 26 |
| `sensor-prod` | 0.47 / 1,223.45 | 4.87 / 681.51 | 4.79% / 70.23% | 3.74 / 116 |

The latest IOPS evidence still supports the previous `io1` to `gp3` conclusion for `odoo-production`: observed I/O peaks remain below the 3,000 IOPS gp3 baseline. The `sensor-prod` `gp2` to `gp3` move remains reasonable and is already represented in `optim.tfvars`.

### Current Production ElastiCache Shape

`smokeprodapiredis` is a Redis 6.2 replication group with cluster mode disabled:

| Node | Type | AZ | Status |
|---|---:|---:|---:|
| `smokeprodapiredis-001` | `cache.t3.medium` | `ap-southeast-2b` | available |
| `smokeprodapiredis-002` | `cache.t3.medium` | `ap-southeast-2c` | available |

Group settings:

- Automatic failover: enabled
- Multi-AZ setting: disabled
- At-rest encryption: enabled
- Transit encryption: disabled
- Snapshot retention: 1 day

Latest 30-day CloudWatch sample, `2026-04-06` to `2026-05-06`:

| Metric | `smokeprodapiredis-001` | `smokeprodapiredis-002` |
|---|---:|---:|
| Database memory usage avg / max | 1.51% / 1.54% | 1.52% / 1.55% |
| Engine CPU avg / max | 0.20% / 0.30% | 0.20% / 0.33% |
| Current connections avg / max | 16.68 / 22 | 5.73 / 8 |

Primary-node network sample:

| Metric | Average | Maximum |
|---|---:|---:|
| `NetworkBytesIn` | 134,978 B/s | 3,447,082 B/s |
| `NetworkBytesOut` | 692,166 B/s | 8,573,246 B/s |

The cache is heavily over-provisioned by memory and CPU. A smaller node type is technically plausible, but the business risk is not just memory: Redis may carry operational session, TTL, lock, queue, or rate-limit state even when the stored footprint is small.

### Application Redis Usage Map

The production backend uses Redis through `src/services/redis/main.redis.ts`, connecting with `REDIS_HOST` and `REDIS_PORT`. The web frontend and SSO provider did not show direct Redis client usage in this pass.

Observed backend usage:

| Area | Key shape / operation | Evidence | Durability meaning |
|---|---|---|---|
| Socket.IO scale-out | `@socket.io/redis-adapter` pub/sub clients | `src/services/socket/main.socket.ts` | Transient fan-out only; no durable app data. |
| Install/manual/scheduled hub command correlation | `<hub_serial>_HUB_VERIFY` with `setEx(..., 20 seconds)` | `src/controllers/users/alarm.controller.ts`, `src/controllers/alarms.controller.ts`, `src/entities/properties.entity.ts`, `src/services/mqtt/subscriber.ts` | Short-lived request/response correlation. Cold start loses only in-flight verification context. |
| Job-scoped alarm log batching | `job_id_<id>` JSON arrays, removed after bulk-create | `src/entities/logs.entity.ts`, `src/controllers/alarms.controller.ts`, `src/services/mqtt/subscriber.ts` | Temporary buffer before Mongo/common/audit log persistence. Loss during a live job could drop pending batched log details. |
| Command/event dedupe and success correlation | `controller_<id>_property_<id>_event_<event>`, `single_controller_<id>_property_<id>_event_<event>` | `src/entities/logs.entity.ts`, `src/services/mqtt/subscriber.ts` | Operational correlation, not a long-term record. |
| Hub power-state notification window | `POWER_STATE_CHANGE` JSON list | `src/services/mqtt/subscriber.ts`, `src/controllers/alarms.controller.ts` | Longer-lived operational state used to decide follow-up DC battery/power notifications. |
| Flapping alert pause/resume window | `<controller>_<alarm>_<index>_<id>_<type>[_status]` | `src/services/mqtt/subscriber.ts` | Suppression window for repeated alert emails. |
| Contractor postcode/radius cache | `contractor_geoJson_data_<userId>` | `src/controllers/properties.controller.ts`, `src/controllers/users/properties.controller.ts`, `src/entities/properties.entity.ts`, `src/entities/settings.entity.ts` | Recomputable cache; stale entries are explicitly removed when contractor settings change. |
| PropertyMe OAuth state | `propertyme_states_<state>` with 120 second TTL | `src/routes/users/v1/properties.routes.ts` | Short-lived OAuth state check. |
| Health check | Redis `PING` included in `/health-check` response | `src/entities/monitoring.entity.ts` | Availability signal; Redis is considered part of app health. |

This is enough to say Redis is not a user-session database and does not need data migration for cold start. However, it is not completely disposable during active workflows: losing it mid-install, mid-batch, or during alert suppression windows can change behavior for in-flight operations.

### Reservation Coverage

Active RDS reservations:

| Product | Class | Count | Offering | Start | State |
|---|---:|---:|---:|---:|---:|
| MySQL | `db.t3.medium` | 3 | No Upfront, hourly recurring charge 0.164 USD | 2025-11-03 | active |
| PostgreSQL | `db.t3.medium` | 2 | No Upfront, hourly recurring charge 0.088 USD | 2025-11-03 | active |

Active ElastiCache reservations:

| Product | Node type | Count | Offering | Start | State |
|---|---:|---:|---:|---:|---:|
| Redis | `cache.t3.medium` | 6 | No Upfront, hourly recurring charge 0.068 USD | 2025-11-03 | active |

Only two RDS DB instances and two cache nodes currently exist in `ap-southeast-2`. Because `sensor-prod` is Multi-AZ, it likely consumes two MySQL `db.t3.medium` reservations. `odoo-production` likely consumes one PostgreSQL `db.t3.medium` reservation. That leaves apparent surplus reservation coverage: roughly one MySQL DB reservation, one PostgreSQL DB reservation, and four Redis `cache.t3.medium` reservations.

## Risks Or Constraints

- AWS access stayed read-only. No remote infrastructure was changed.
- Terraform prod files were not edited because infra file edits require explicit approval.
- Instance-class right-sizing may not reduce cash spend while the existing one-year reservations remain active.
- `sensor-prod` CPU has occasional spikes, so RDS instance downsizing is not recommended from this evidence alone.
- Redis memory and CPU are low, but a safe downsize still needs application-level confirmation of Redis durability expectations and latency sensitivity.

## Cost Optimization Opportunities

### 1. Move `odoo-production` RDS storage from `io1` to `gp3`

- **Priority:** highest RDS quick win
- **Terraform target:** `aws_db_instance.odoo_production` in `code/infra/terraform/environments/prod/generated_data.tf`
- **Current:** `storage_type = "io1"`, `iops = 3000`, `allocated_storage = 100`
- **Proposed:** `storage_type = "gp3"` with gp3 baseline 3,000 IOPS / 125 MiB/s
- **Evidence:** latest read/write IOPS remain below 3,000 IOPS baseline.
- **Expected impact:** approximately USD 320/month from removing provisioned IOPS charges, based on May Cost Explorer `APS2-RDS:PIOPS` usage projected to a full month. This aligns with the earlier deep analysis estimate of about USD 317/month.
- **Tradeoff:** production RDS storage modification; requires maintenance-window planning, baseline capture, Terraform plan review, and post-change health checks.
- **Local proposal status:** approved for local Terraform preparation on 2026-05-06 by "ok, lets apply RDS"; Codex must not run the remote AWS/Terraform apply from this repository.

### 2. Move `sensor-prod` RDS storage from `gp2` to `gp3`

- **Priority:** safe cleanup / quality improvement
- **Terraform target:** modeled through `rds_storage_type = "gp3"` and `rds_storage_throughput = 125`; a focused `code/infra/terraform/environments/prod/rds-storage-gp3.tfvars` file was added so this can be reviewed separately from the broader `optim.tfvars` bundle.
- **Evidence:** latest IOPS peaks remain below gp3 baseline; prior RDS deep analysis already marked the migration safe.
- **Expected impact:** small direct monthly saving because this DB is only 20 GiB, but gp3 gives predictable baseline performance.
- **Tradeoff:** production RDS storage modification; keep Multi-AZ enabled.

### RDS local Terraform proposal status

Prepared locally on 2026-05-06:

- `code/infra/terraform/environments/prod/generated_data.tf`
  - `aws_db_instance.odoo_production.storage_type`: `io1` -> `gp3`
  - `aws_db_instance.odoo_production.iops`: `3000` -> omitted
  - `aws_db_instance.odoo_production.storage_throughput`: `0` -> omitted
- `code/infra/terraform/environments/prod/rds-storage-gp3.tfvars`
  - `rds_storage_type = "gp3"`
  - `rds_storage_throughput = null`
  - `rds_iops = null`

Validation and targeted plan:

- `terraform validate -no-color`: success; only pre-existing Lambda `ignore_changes` warnings.
- `terraform plan -lock=false -input=false -no-color -target=aws_db_instance.odoo_production -target=aws_db_instance.sensor_prod -var-file=rds-storage-gp3.tfvars -detailed-exitcode`:
  - Plan exit code `2`, expected because changes are present.
  - `Plan: 0 to add, 2 to change, 0 to destroy.`
  - Both changes are in-place RDS updates:
    - `odoo-production`: `io1` -> `gp3`, omit explicit IOPS/throughput for 100 GiB PostgreSQL gp3
    - `sensor-prod`: `gp2` -> `gp3`, omit explicit IOPS/throughput for 20 GiB MySQL gp3
  - Warnings: deprecated S3 backend `dynamodb_table`, targeted plan warning, and pre-existing Lambda `ignore_changes` warnings.

External apply attempt on 2026-05-06 failed with RDS validation errors:

- `odoo-production`: AWS rejected explicit IOPS/throughput for PostgreSQL gp3 storage under 400 GiB.
- `sensor-prod`: AWS required explicit IOPS for gp3.

The local Terraform proposal was corrected after that failed apply: `odoo-production` now omits explicit IOPS/throughput, while `sensor-prod` now omits explicit IOPS/throughput for the same sub-400 GiB gp3 rule.

Second external apply attempt on 2026-05-06 partially succeeded:

- `odoo-production` Terraform modification completed externally.
- Read-only AWS check immediately afterward showed `odoo-production` still reporting current `io1` with pending modified values for `StorageType=gp3`, `Iops=3000`, `StorageThroughput=125`. Treat this as pending AWS storage modification until a later read-only check shows current `storageType=gp3`.
- `sensor-prod` failed again because AWS rejected explicit IOPS/throughput for 20 GiB MySQL gp3. The local varfile was corrected to use `null` for both values.
- Corrected sensor-only read-only plan:
  - `terraform plan -lock=false -input=false -no-color -target=aws_db_instance.sensor_prod -var-file=rds-storage-gp3.tfvars -detailed-exitcode`
  - `Plan: 0 to add, 1 to change, 0 to destroy.`
  - Planned update: `sensor-prod.storage_type` `gp2` -> `gp3`; no explicit IOPS or storage throughput changes.

Third external apply attempt on 2026-05-06 was reported as applied by the user/operator:

- Read-only AWS verification immediately afterward initially showed both storage changes pending.
- User/operator then applied both pending modifications immediately.
- Follow-up read-only AWS verification showed:
  - `odoo-production`: current `storageType=gp3`, `Iops=3000`, `StorageThroughput=125`, no pending modified values, status `storage-optimization`
  - `sensor-prod`: current `storageType=gp3`, `Iops=3000`, `StorageThroughput=125`, no pending modified values, status `storage-optimization`
- RDS events showed both storage modifications finished applying. `sensor-prod` CloudWatch `DatabaseConnections` stayed flat at 2 through the checked window.
- Terraform prod defaults were updated after verification so the current prod snapshot matches the live gp3 RDS storage shape.

### 3. Do not right-size RDS instance classes yet

- **Reason:** active `db.t3.medium` reservations run until roughly 2026-11-03, and current CPU samples show occasional production spikes.
- **Expected impact:** downsizing now may create risk without removing the active reservation commitment.
- **Better action:** plan reservation renewal hygiene before 2026-11-03. Do not renew surplus `db.t3.medium` reservations unless matching workloads still exist.

### 4. Treat Redis node right-sizing as a renewal-timed change

- **Current on-demand reference prices in Sydney:**
  - `cache.t3.medium`: 0.100 USD/node-hour
  - `cache.t3.small`: 0.050 USD/node-hour
  - `cache.t3.micro`: 0.025 USD/node-hour
  - `cache.t4g.small`: 0.048 USD/node-hour
  - `cache.t4g.micro`: 0.024 USD/node-hour
- **Technical evidence:** memory max is about 1.55%; engine CPU max is about 0.33%.
- **Application evidence:** Redis is used for transient backend coordination, Socket.IO pub/sub, in-flight alarm/install state, postcode/radius cache, PropertyMe OAuth state, and some longer-lived operational alert windows. It is not the user-session database.
- **Reservation constraint:** six active `cache.t3.medium` reservations remain in force while only two cache nodes exist. Reducing the production nodes now may not reduce cash spend until the reservations expire.
- **Recommended posture:** technically reasonable target is smaller than `cache.t3.medium` based on memory/CPU. A conservative first production target would be `cache.t4g.small` or `cache.t3.small` with the same two-node replication-group shape, then reassess after live key/traffic inspection. Do not remove Redis or drop to one node for production. Cash savings may still be blocked until the existing reservations expire or can be corrected.

## Recommended Next Steps

1. Seek explicit approval for a focused Terraform proposal to change only `odoo-production` RDS storage from `io1` to `gp3`.
2. Run a read-only Terraform plan for prod before any approved apply and confirm no destroys.
3. Apply `sensor-prod` `gp2` to `gp3` either as part of the existing `optim.tfvars` production optimisation bundle or as a separate, reviewable storage-only change.
4. Add a reservation review reminder before 2026-11-03 so the surplus RDS and Redis reservations are not renewed by habit.
5. Before any Redis downsize, verify live Redis key categories, TTL profile, eviction policy, failover behavior, and whether any workflow treats Redis as durable state.

## Open Questions

- Are the surplus RDS and ElastiCache reservations intentional coverage for workloads that were recently removed, or accidental over-commitment after sandbox/non-prod cleanup?
- Is there any finance or AWS account-management path to adjust the active cache reservation commitment before 2026-11-03?
- Which Redis key patterns are durable enough to require snapshot/restore confidence before a node-size change?
