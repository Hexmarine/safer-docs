# Rough Migration Plan — Source Account to New JV Account

- Date: 2026-04-10
- Scope: high-level phased migration plan for moving the production SensorGlobal platform from AWS account `747293622182` to the new joint venture AWS account; includes deeper investigations required before each phase can be executed
- Related systems: all production systems — EMQX MQTT broker, Kafka, EC2 API cluster, RDS MySQL and PostgreSQL, MongoDB, ElastiCache Redis, CloudFront, ALB, Route 53, ACM, WAF, KMS, IAM, KORE SIM management, PropertyMe, Twilio, SendGrid, Odoo, New Relic

## Objective

Provide a rough but actionable migration plan with phases, sequencing dependencies, and a list of targeted deeper investigations that must be completed before each phase can be safely executed. This document is the planning anchor; it will be refined as investigation results come in.

## Sources Used

- Repository investigations:
  - `docs/investigations/2026-04-09-aws-account-baseline.md`
  - `docs/investigations/2026-04-09-ap-southeast-2-architecture-baseline.md`
  - `docs/investigations/2026-04-09-prod-external-account-migration-evaluation.md`
  - `docs/investigations/2026-04-09-platform-architecture-and-data-flow.md`
  - `docs/investigations/2026-04-09-mqtt-device-communications.md`
  - `docs/investigations/2026-04-09-alarm-event-traces.md`
  - `docs/investigations/2026-04-10-cost-deep-dive.md`
  - `docs/investigations/2026-04-10-prod-user-device-migration-readiness.md`

## Updated Findings (2026-04-10, second investigation pass)

The following was confirmed via direct AWS inspection in addition to prior investigation notes:

- **Secrets management confirmed**: All application configuration for every environment lives in AWS Secrets Manager (not SSM Parameter Store). Named secrets: `sensor-prod`, `sensor-prod-sso`, `sensor-qa`, `sensor-qa-sso`, `sensor-dev-sso`, `sensor-sandbox`, `sensor-sandbox-sso`, `sensor-uat`, `sensor-uat-sso`, `smoke-dev`, `local`, `local-dev-sso`. SSM Parameter Store has minimal use — only CloudWatch agent configs (`AmazonCloudWatch-MemoryConfig`, `AmazonCloudWatch-linux`) and Inspector path lists. No application credentials in SSM.
- **Prod Redis confirmed + cold-start safe** (INV-03): Two nodes `smokeprodapiredis-001` / `002`, `cache.t3.medium` pair, parameter group `smokeapi`, `volatile-lru` eviction. All application keys use `setEx` (TTL) — no long-lived data. User sessions are in MySQL, NOT Redis. **Cold-start at cutover is safe.**
- **MongoDB is Atlas (cloud-managed)** (INV-02): `MONGO_DB_URL` points to `sensor-prod.vr48f.mongodb.net` — a **MongoDB Atlas** managed cluster, NOT a self-hosted EC2 instance. Atlas Online Archive configured at 90 days. Migration approach: Atlas cluster-to-cluster (Atlas Live Migration or mongodump/restore to a new Atlas cluster in the JV entity's Atlas org). No MongoDB EC2 instance exists in the prod VPC.
- **All 6 prod portals share ONE S3 bucket** (INV-04): `smokealarmprod-admin` (24 MB, 592 objects) → ONE CloudFront distribution. Frontend migration is a re-deploy, not a data migration. Main app asset bucket `smokeapibucket` = 3.4 GB and must be synced cross-account.
- **Cron server discovered**: `smoke-prod-api-cron-server` (t2.micro, `i-0a1cd47cfadedf035`) — dedicated scheduled-job server. Must be added to Phase 3 rebuild.
- **WordPress separate from Odoo**: `wordpress-prod` (t3.medium, behind ALB, no public IP) — public website, distinct from Odoo CRM. Must be added to Phase 3.
- **KORE/ONO SIM IPs in EMQX SG confirmed** (INV-01 ✅): ~20 CIDRs for port 1883 AND port 18756 must be exactly reproduced. Port 18756 = **MQTT over TLS** (custom listener, confirmed via SSH) — primary hub port with 16 of 21 connected clients. Personal dev IP `116.73.36.32/32` ("lokendra") has ALL-traffic permission — do NOT carry to new account. **🚨 TLS cert on port 18756 expired 2026-04-08 — renew immediately.**
- **All environments share keychain ALB**: DNS variants `keychain.*`, `keychain-development.*`, `keychain-qa.*`, `keychain-sandbox.*` all point to the same ALB. New account keychain must be live before any keychain DNS record is updated.
- **Domains in Route53 Registrar** (INV-06): Route53-to-Route53 transfer — no external registrar involvement. Several near-expiry in 2026: `sensorglobal.au`/`.net` (Jun), `.co.uk`/`.com.au` (Jul), `sensorinsure.com`/`.com.au` (Sep). AutoRenew is ON but verify payment method before closing source account.
- **DMARC `p=none`**: Not enforcing — set `p=quarantine` in new account email setup.
- **Zabbix monitoring**: `zabbix_monitoring` IAM user exists — Zabbix is active alongside New Relic. Include in monitoring migration.
- **Bitrise CI/CD** (INV-07): CodeDeploy is BLUE_GREEN (WITH_TRAFFIC_CONTROL) for prod APIs. ALBs + target groups must be pre-configured before first CodeDeploy run in new account. 16 CodePipeline pipelines including `Smoke-production-admin` (Angular SPA → S3 → CloudFront invalidation).
- **IAM access keys in secrets**: `sensor-prod` contains `AWS_ACCESS_KEY`/`AWS_SECRET_KEY` for IAM user auth. **Do NOT carry these to the new account** — replace with EC2 instance roles and proper IAM.
- **All non-prod envs have parallel Secrets Manager secrets**: new account should replicate the same secret naming pattern per environment.

---

## Migration Overview

The migration has seven phases plus a pre-flight investigation phase. **Because the deadline is ASAP**, Phases 1, 2, and 3 must run in parallel — do not wait for data migration to complete before standing up servers. Phase 4 (MQTT cutover) is isolated as the highest-risk single event. Phases 5–7 follow in sequence.

**Rough total critical path: 6–9 weeks.** The biggest wildcard is KORE re-registration — if not started immediately, it can add 2–4 weeks to the overall timeline.

### Phase schedule overview

| Phase | Scope | Rough effort | Notes |
|---|---|---|---|
| 0 | Investigations | 1–2 weeks | Can parallelize all 8 investigations; blocked until OpenVPN + target account access exists |
| 1 | New account foundation | 1–2 weeks | Starts as soon as target account credentials are available; runs in parallel with Phase 3 |
| 2 | Data migration | 3–5 days prep + 2–4 hr maintenance window | Execution is short; prep (create empty DBs, network paths) overlaps with Phase 3 |
| 3 | Application rebuild | 2–3 weeks | Longest phase; runs in parallel with Phase 1 once VPC/networking exists |
| 4 | MQTT + Kafka cutover | 1 maintenance window (2–4 hr) + 48 hr pre-work | Highest-risk isolated event |
| 5 | DNS + certificate cutover | 1 maintenance window (2–4 hr) + 48 hr pre-work | Follows Phase 4 stability confirmation |
| 6 | Integrations + validation | 1–2 weeks | KORE is the wildcard; start engagement in Week 1, not here |
| 7 | Decommission | 2–3 weeks | Minimum 2-week soak after cutover before terminating source resources |

### Critical path (parallelized for ASAP)

```
Week 1–2:  Phase 0 (investigations) ──────────────────────────────┐
           + KORE vendor engagement START (do not wait)            │
                                                                   ▼
Week 2–4:  Phase 1 (account setup) ──────┐                        │
           Phase 3 (rebuild) starts ─────┤◄─────── depends on Phase 0 + Phase 1 VPC
           Phase 6 integrations begin ───┘ (KORE, PropertyMe, SendGrid, Twilio — long lead)

Week 3–5:  Phase 3 (compute rebuild) continues
           Phase 2 prep (empty DBs, network paths for snapshot)

Week 5–6:  Phase 2 execution (maintenance window — snapshot+restore)
           Phase 5 (parallel validation using mqtt-prototype simulator)

Week 6:    Phase 4 (MQTT+Kafka DNS cutover — maintenance window)
           Phase 5 (DNS + cert cutover — maintenance window, same night or next)

Week 7–9:  Phase 6 (integrations validation)
           Phase 7 (decommission — 2-week soak then cleanup)
```

---

## Phase 0: Deeper Investigations (Pre-requisite for All)

> **Summary:** 8 targeted investigations to close the unknowns that block every other phase. All 8 have been completed. INV-01 was completed via SSH to `Emqx-prod` on 2026-04-10.
> **Status as of 2026-04-10:** INV-01 ✅, INV-02 ✅, INV-03 ✅, INV-04 ✅, INV-05 ✅, INV-06 ✅, INV-07 ✅, INV-08 ✅
> **Full results:** `docs/investigations/2026-04-10-inv-results.md`

### INV-01 — EMQX broker configuration and hub credential model ✅

**Status: Complete (SSH to `Emqx-prod` obtained 2026-04-10).**

**Findings:**
- **EMQX version 5.8.7** — single node, `discovery_strategy = manual`, no clustering
- **21 connected clients**: 16 on port 18756 (TLS MQTT), 5 on port 1883 (plain TCP)
- **Port 18756 = MQTT over TLS (custom port)** — this is the primary hub connection port. Custom `*.sensorglobal.com` wildcard cert. Server-side TLS only (`verify = verify_peer`, `fail_if_no_peer_cert = false`).
- **Port 1883** = plain TCP MQTT — used by API servers connecting from the internal VPC
- **🚨 TLS cert on port 18756 EXPIRED 2026-04-08** — `notAfter: Apr 8 23:59:59 2026 GMT`. Hub firmware ignores expiry, but the new broker must have a valid cert from day one
- **Authentication**: built-in database, username/password, **plaintext passwords** (`salt_position = disable`)
- **Credential export**: `sudo emqx_ctl data export` → `/var/lib/emqx/backup/` — includes all auth credentials; import with `emqx_ctl data import` on new instance
- **ACL: effectively open** — final rule is `{allow, all}`, `no_match = allow`. No per-device or per-topic restrictions. The `acl.conf` comment warns to change to `{deny, all}` for production — never done
- **Security risk:** `116.73.36.32/32` ("lokendra") has ALL-traffic access on SG — do NOT carry to new account

**Output:** `docs/investigations/2026-04-10-inv-results.md` — INV-01 section ✅ complete.

---

### INV-02 — MongoDB ✅ COMPLETE

**Key finding: MongoDB is Atlas (cloud-managed), NOT a self-hosted EC2 instance.**

- Cluster: `sensor-prod.vr48f.mongodb.net` (MongoDB Atlas)
- Database: `sensorproddb`, user: `sensorproddb_usr`
- Atlas Online Archive configured at 90-day data lifecycle
- No MongoDB EC2 in the prod VPC — never was

**Migration approach updated:** Use Atlas Live Migration (zero-downtime) or mongodump/restore (maintenance window) to a new Atlas cluster in the JV entity's Atlas organization. See Phase 2.2.

**Output:** `docs/investigations/2026-04-10-inv-results.md` — INV-02 section.

---

### INV-03 — ElastiCache Redis ✅ COMPLETE

**Key finding: Cold-start at cutover is safe.**

- Cluster: `smokeprodapiredis-001` (primary) + `002` (replica), `cache.t3.medium` each
- Engine: Redis 6.2.6, param group `smokeapi`: `maxmemory-policy=volatile-lru`, `timeout=0`
- All application keys use `setEx` (SETEX with TTL) — no long-lived data in Redis
- User sessions are in MySQL `tbl_sessions`, not Redis

**Migration approach:** Provision fresh cluster in new account (same spec), recreate `smokeapi` parameter group. No data migration.

**Output:** `docs/investigations/2026-04-10-inv-results.md` — INV-03 section.

---

### INV-04 — S3 and CloudFront ✅ COMPLETE

**Key findings:**
- All 6 prod portals served from ONE S3 bucket (`smokealarmprod-admin`, 24 MB) via ONE CloudFront distribution — frontend migration is a re-deploy, not a data copy
- `smokeapibucket` (3.4 GB, 31,743 objects) = main app document/asset storage — must be synced cross-account
- 70+ total buckets; most are QA/dev/logs — prod migration needs only `smokeapibucket` + `smokealarmprod-admin`

**Output:** `docs/investigations/2026-04-10-inv-results.md` — INV-04 section.

---

### INV-05 — IAM, Secrets, and Credentials ✅ COMPLETE

**Key findings:**
- 24 IAM users: staff, Appinventiv dev team (decision needed: does JV retain Appinventiv?), `bitrise`, `zabbix_monitoring`, service accounts
- One prod EC2 instance profile: `smoke-alaram-api-logs-role`
- `sensor-prod` secret contains ~100 key-value pairs including **hardcoded IAM access keys** (`AWS_ACCESS_KEY`/`AWS_SECRET_KEY`) — replace with IAM roles in new account
- Full third-party integration credential inventory: KORE, Twilio, SendGrid (2 keys), PropertyMe, PropertyTree, Console Cloud, Sentry, LaunchDarkly, Google (reCAPTCHA, Maps, Firebase), Odoo, NEXU, Corpsure
- **Zabbix monitoring** is in use (in addition to New Relic, which is being dropped)

**Output:** `docs/investigations/2026-04-10-inv-results.md` — INV-05 section.

---

### INV-06 — DNS and Registrar ✅ COMPLETE

**Key findings:**
- All 14 domains registered via **AWS Route 53 Registrar** — Route53-to-Route53 transfer, no external registrar
- `sensorglobal.com` has 119 records; EMQX and Kafka use direct A records (not ALBs)
- All environments share ONE keychain ALB — `keychain.*` DNS must not change until new keychain is live in new account
- **6 domains expire mid-2026** (`sensorglobal.au`, `.net`, `.co.uk`, `.com.au`, `sensorinsure.com`, `sensorinsure.com.au`) — AutoRenew is on but payment method must be valid before source account is closed
- **DMARC `p=none`** — should be strengthened to `p=quarantine` in new account

**Recommended DNS strategy:** Keep hosted zone in source account during cutover. Update A/CNAME records record-by-record as services migrate. Transfer zone and domain registrations to new account after soak period.

**Output:** `docs/investigations/2026-04-10-inv-results.md` — INV-06 section.

---

### INV-07 — CI/CD and Deployment Automation ✅ COMPLETE

**Key findings:**
- **Bitrise** is the CI/CD build platform (IAM user `bitrise` exists; no `bitrise.yml` in repo — config is in Bitrise cloud)
- CodeDeploy: 11 applications; prod API uses **BLUE_GREEN** with `WITH_TRAFFIC_CONTROL` — ALBs and target groups must be pre-configured before first CodeDeploy run
- 16 CodePipeline pipelines including `Smoke-production-admin` (Angular SPA → S3 → CloudFront invalidation)
- `appspec.yml` is simple and portable; account-specific values (S3 bucket, CodeDeploy role ARN) are in CodeDeploy configuration, not the AppSpec

**What must be recreated:** CodeDeploy apps + deployment groups (BLUE_GREEN), CodePipeline `Smoke-production-admin`, Bitrise IAM user/OIDC config targeting new account.

**Output:** `docs/investigations/2026-04-10-inv-results.md` — INV-07 section.

---

### INV-08 — Active User and Device Headcount ✅ COMPLETE

**Key findings:**
- RDS `DatabaseConnections` 7-day: avg ~4.4, peak ~64 — low connection pressure, healthy
- EMQX CloudWatch logs empty — cannot determine connected device count without SSH
- API request rate: ~8,300–8,600 req/hr average (mostly automated background traffic)
- Current prod sizing (`t3.medium` × 5 API, `c5.xlarge` EMQX) is appropriate for observed load

**Sizing recommendation:** Match current instance types in new account. Start with 3× `t3.medium` API servers if cost is a concern and scale up.

**Output:** `docs/investigations/2026-04-10-inv-results.md` — INV-08 section.

---

## Phase 1: Target Account Baseline

> **Summary:** One-time account setup that everything else depends on. Get VPC, IAM, KMS, ACM certs, and CloudTrail stood up correctly the first time — getting this wrong causes cascading rework. Aim to run this in parallel with INV investigations from the moment target account credentials arrive.
> **Rough effort:** 1–2 weeks. Blocked until: target account provisioned, INV-05 (secrets/IAM) and INV-06 (DNS) complete.

**Depends on:** INV-05 (IAM/secrets), INV-06 (DNS strategy), basic account provisioning decisions from JV stakeholders.

**Goal:** Stand up the new AWS account with the governance, networking, and security baseline needed to host production workloads.

### 1.1 — Account and organization setup
- Confirm the new JV account ID and whether it joins a new AWS Organization or is standalone.
- Establish governance guardrails (SCPs if org-based, Config rules, CloudTrail, Security Hub) equivalent to or better than current account's posture.
- Enable `ap-southeast-2` as the primary region (matching the current account).
- Set up a break-glass IAM admin and a read-only investigator role.

### 1.2 — VPC and networking
- Create a production VPC in `ap-southeast-2` mirroring the current prod VPC structure:
  - 3 public subnets (`10.0.1.0/24`, `10.0.2.0/24`, `10.0.3.0/24`)
  - 3 private application subnets (`10.0.4.0/24`, `10.0.5.0/24`, `10.0.6.0/24`)
  - 3 private RDS subnets
  - Internet gateway and one public NAT gateway
- Set up VPC peering or Transit Gateway if multi-VPC is needed later for non-prod.
- Set up an OpenVPN or AWS Client VPN endpoint for operational access.
- Decide whether to replicate the current multi-environment VPC pattern (prod + qa + dev) or start with prod-only and add environments later.

### 1.3 — KMS and encryption baseline
- Create customer-managed KMS keys for: RDS at-rest encryption, S3 bucket encryption, CloudTrail logs, future Security Hub and Config storage.
- Do not attempt to copy the source account's KMS keys — they are account-bound. All new keys will have new ARNs, and this will be reflected in the rebuilt services.

### 1.4 — IAM roles and instance profiles
- Create EC2 instance profiles for: API servers, SSO, MQTT broker, Kafka, Odoo, WordPress, cron server.
- Create service roles for: CodeDeploy, CloudWatch agent, S3 access patterns used by the application.
- Create operational roles for the migration team and ongoing DevOps access.

### 1.5 — Monitoring baseline

**Confirmed: CloudWatch only.** New Relic will not be carried forward. No Metric Stream, no Firehose delivery stream. Remove all New Relic agent references from EC2 user-data and appspec scripts.

**CloudWatch agent setup:**
- Install CloudWatch agent on every EC2 host using the `AmazonCloudWatch-linux` SSM parameter pattern (matches current source account).
- Configure disk and memory metrics (custom namespace `CWAgent`) — these are not available by default.
- Configure PM2 log file paths to stream to CloudWatch log groups.

**Log retention policy — apply to every log group at creation time:**

All log groups in the new account must have an explicit retention set. Never allow `None` (no expiry). The current source account has ~133 GB of prod VPC flow logs and ~133 GB of prod API PM2 stdout accumulated at 365-day retention — this is the noise and cost baseline we are avoiding.

| Log group pattern | Retention | Rationale |
|---|---|---|
| `smoke-api-prod-pm2-out-log` | **30 days** | Very high volume application stdout; 30 days is sufficient for debugging |
| `smoke-api-prod-pm2-error-log` | **90 days** | Error logs; lower volume, longer useful window |
| `sso-prod-api-logs` | **90 days** | Auth logs; some compliance value |
| `sso-prod-api-error` | **90 days** | Auth errors |
| `smoke-vpc-flow-log` (prod VPC) | **90 days** | Security/network investigation; 133 GB at 365 days in source — 90 days cuts stored volume by ~75% |
| `aws-waf-logs-prod` | **90 days** | WAF request logs; 63 GB at 365 days in source; 90 days sufficient for incident review |
| Redis engine/slow logs | **30 days** | Performance debugging only; not useful after a month |
| Lambda function logs | **30 days** | Short-lived; rarely needed after a month |
| CodeBuild / CI logs | **30 days** | Build logs; not needed after a few weeks |
| SNS delivery logs | **90 days** | Notification delivery audit trail |
| CloudTrail (via log group) | **365 days** | Compliance and security audit — retain longer |
| GuardDuty, Security Hub findings | **365 days** | Security audit trail |
| Cron server logs | **30 days** | Job output; low long-term value |
| OpenVPN / VPN access logs | **90 days** | Access audit |

**Cost context:** CloudWatch Logs storage in `ap-southeast-2` is ~$0.03/GB/month. The current source account has accumulated roughly **560 GB** across prod VPC flow logs, prod API stdout, QA, sandbox, dev, and WAF. At 90-day retention instead of 365 days, the new account's steady-state storage would be approximately 25% of the current level for equivalent traffic.

**Do not create:**
- `/aws/kinesisfirehose/PUT-S3-*` (was New Relic delivery)
- `hivemq-prod-*` (HiveMQ is not used — EMQX is; these are stale groups in source)
- `sonarqube-*` (SonarQube not in scope)
- Any AmazonMQ broker groups (AmazonMQ not in use in the new platform)



## Phase 2: Data Layer Migration

> **Summary:** The only truly irreversible step. A maintenance window snapshot+restore keeps it simple and safe: take RDS and MongoDB snapshots, restore to new account, verify row counts and integrity checksums before any traffic moves. Prep work (empty DB infra in new account, network paths, key sharing) can run in parallel with Phase 3 during the weeks before the maintenance window.
> **Rough effort:** 3–5 days prep (overlaps with Phase 3) + a 2–4 hour maintenance window for execution. Blocked until: Phase 1 (network + KMS), INV-02 (MongoDB host and volume), INV-03 (Redis eviction policy).

**Depends on:** Phase 1 (network and encryption baseline), INV-02 (MongoDB), INV-03 (Redis).

**Confirmed: a maintenance window is acceptable.** This simplifies all data migration steps. Snapshot+restore approaches are preferred over live replication, as they are simpler, lower-risk, and sufficient for an off-peak cutover window.

**Goal:** Get all stateful production data into the new account within the maintenance window.

### 2.1 — MySQL (sensor-prod) — confirmed approach: snapshot + restore

**Confirmed sizing:** `db.t3.medium`, 20 GB gp2, **11.7 GB used**, MultiAZ ✅, encrypted ✅.

Steps:
1. Take an RDS snapshot of `sensor-prod` (Multi-AZ — snapshot taken from standby, no downtime on primary).
2. **Share KMS key cross-account first** (or re-encrypt snapshot with target account's KMS key) — `sensor-prod` storage is encrypted; unencrypted cross-account snapshot sharing is not possible.
3. Share the snapshot cross-account to the target account.
4. Restore into a new `db.t3.medium` Multi-AZ RDS MySQL 8.0 instance in the target account with **gp3** storage (upgrade from gp2 — cheaper with same or better performance).
5. **Recreate custom parameter group `prod-mysql-8-parameter-group`** with:
   - `group_concat_max_len = 100000`
   - `log_bin_trust_function_creators = 1`
   - Apply parameter group to the restored instance before starting the application.
6. Validate row counts on key tables (`tbl_admins`, `tbl_alarms`, `tbl_properties`, `tbl_jobs`, `tbl_sessions`, `tbl_alarm_alerts`) before enabling API. `tbl_alarm_alerts` is the P1-critical alert pipeline table — any row count discrepancy here means the notification pipeline will be broken from day one.

**Estimate:** Snapshot restore for 11.7 GB ≈ 15–25 minutes. KMS key sharing setup = 30 minutes one-time prep.

### 2.2 — MongoDB Atlas cluster migration

> **Context:** MongoDB is Atlas-managed (`sensor-prod.vr48f.mongodb.net`), not a self-hosted EC2 instance. No MongoDB EC2 to snapshot. Atlas Online Archive is configured at 90 days.

**Two options (choose based on acceptable downtime):**

**Option A — Atlas Live Migration (zero downtime):**
1. Create a new MongoDB Atlas cluster in the new JV entity's Atlas organization (same tier as source — check Atlas billing for current tier).
2. Use Atlas Live Migration (available in Atlas UI under "Migrate Data to This Cluster") to set up continuous replication from the source cluster to the new cluster.
3. Monitor lag until it drops to seconds, then cut the application over (update `MONGO_DB_URL` in the new account's `sensor-prod` secret).
4. Terminate the source cluster after soak period.

**Option B — mongodump / mongorestore (maintenance window):**
1. Create new Atlas cluster in JV Atlas org.
2. During the maintenance window (API in maintenance mode): `mongodump --uri "$MONGO_DB_URL" --out /tmp/mongodump/`.
3. `mongorestore --uri "$NEW_MONGO_DB_URL" /tmp/mongodump/`.
4. Validate document counts per collection.
5. Update `MONGO_DB_URL` in new `sensor-prod` secret.

**Atlas Online Archive:** Configure Atlas Online Archive in the new Atlas project with the same 90-day policy. The archive endpoint URL will change — update `MONGO_DB_ARCHIVE_URL` in the new `sensor-prod` secret.

**Action required before migration:** Get access to the Atlas console (atlas.mongodb.com) to confirm cluster tier, region, and billing. The Atlas cluster is billed separately from AWS and may be under a different account login.

### 2.3 — Odoo PostgreSQL (odoo-production)

**Confirmed sizing:** `db.t3.medium`, 100 GB io1 (3000 IOPS), **10 GB used**, no MultiAZ, encrypted ✅.

Steps:
1. Stop the Odoo application server cleanly before snapshot (Odoo has no Multi-AZ; stopping app ensures clean state).
2. Take RDS snapshot → share KMS key cross-account → share snapshot.
3. Restore in target account as `db.t3.medium` with **gp3 storage** (not io1):
   - Set gp3 IOPS to 3000 (matches current io1 performance) at ~65% lower cost (~$115/month vs ~$330/month).
   - Allocated size: 100 GB (carry forward to avoid post-restore storage resize complexity).
4. Apply `default.postgres14` parameter group (no custom params — nothing to carry forward).
5. After restore, validate Odoo starts against the new DB endpoint.
6. **MultiAZ decision:** Current source has no Multi-AZ. Recommend keeping Single-AZ in new account unless Odoo uptime requirements change.

**Estimate:** Snapshot restore for 10 GB ≈ 15–25 minutes (RDS allocates time based on allocated size, not used; 100 GB io1 restore may take 30–45 min). Plan 1-hour Odoo maintenance window.

### 2.4 — ElastiCache Redis

> **Finding (INV-03):** Cold-start is confirmed safe. All Redis keys use `setEx` (TTL-based via `setDataInRedisWithExpireTime`). User sessions are stored in MySQL `tbl_sessions`, not Redis. No warm migration needed.

- Provision a fresh ElastiCache Redis cluster in the new account: 2× `cache.t3.medium`, Redis 6.2.6 or later, replication enabled.
- Create a custom parameter group matching `smokeapi`: `maxmemory-policy = volatile-lru`, `timeout = 0`.
- Update `REDIS_HOST` in the new account's `sensor-prod` secret.
- Do NOT attempt to snapshot or copy existing Redis data.


### 2.5 — S3 content migration

> **Finding (INV-04):** Two buckets matter for data migration. The Angular SPA bucket (`smokealarmprod-admin`, 24 MB) is a build artifact — just redeploy. The main app bucket (`smokeapibucket`, 3.4 GB) contains user documents and assets and must be synced.

**`smokeapibucket` (3.4 GB, 31,743 objects) — must migrate:**
- Set up a cross-account IAM role in the target account with read/write access to the new bucket.
- Run `aws s3 sync s3://smokeapibucket s3://NEW-BUCKET-NAME --source-region ap-southeast-2 --region ap-southeast-2` via an EC2 instance in the source account (or a Lambda with cross-account role).
- Run a second sync pass immediately before cutover to pick up any delta.
- Update `AWS_S3_BUCKET` in the new `sensor-prod` secret.

**`smokealarmprod-admin` (24 MB) — redeploy instead:**
- In the new account, create the new S3 bucket and CloudFront distribution.
- Trigger the `Smoke-production-admin` CodePipeline to build and deploy the Angular SPA.
- No data to migrate — this is a compiled build artifact.
- Update `AWS_S3_URL` (CloudFront URL) in the new `sensor-prod` secret.

**Log/audit buckets:** Do not migrate. Set up new buckets in new account; source account buckets are retained for the soak period.

---

## Phase 3: Application Tier Rebuild

> **Summary:** The longest and most manual phase. Every service (EMQX, API servers, Redis, Odoo, CloudFront, WAF, ALBs) must be stood up, configured, and smoke-tested in the new account before any production traffic moves. The deployment process should be automated as much as possible (use INV-07 findings). EMQX configuration must be verified against hub connectivity using `mqtt-prototype` simulator before any DNS change.
> **Rough effort:** 2–3 weeks. Runs in parallel with Phase 1 (once VPC exists) and Phase 2 prep. Blocked until: Phase 1 VPC/networking, INV-01 (EMQX config), INV-07 (deployment automation), INV-05 (secrets structure).

**Depends on:** Phase 1 complete, INV-07 (deployment automation), INV-05 (secrets/credentials).

**Goal:** Rebuild all compute-tier services in the new account so they can be tested against the replicated data before any traffic is shifted.

### 3.1 — EMQX MQTT broker (new instance, pre-loaded config)

- Launch a `c5.xlarge` EC2 instance in the new prod VPC (public subnet, public IP will become `mqtt.sensorglobal.com` after DNS flip).
- Install **EMQX 5.8.7** (same version as production — confirmed via SSH on 2026-04-10).
- **Export credentials from source broker:**
  ```bash
  sudo emqx_ctl data export
  # Produces a .tar.gz in /var/lib/emqx/backup/
  # Transfer to new instance and import:
  sudo emqx_ctl data import <exported-file.tar.gz>
  ```
  This exports all auth credentials (built-in database, plaintext passwords) plus cluster config. **Only 1 credential exists** — confirmed via `emqx eval "ets:tab2list(emqx_authn_mnesia)."` — and it is the same UUID pair already in the `sensor-prod` Secrets Manager secret. The `data export` file is primarily useful for restoring the ACL, dashboard admin, and API keys; the credential itself does not need the export.
- **Configure TLS listener on port 18756** (primary hub port):
  - Issue a new **valid** `*.sensorglobal.com` TLS certificate (the current cert expired 2026-04-08; use ACM + ALB, or Let's Encrypt via `certbot` on the EC2 instance)
  - Reproduce the `ssl:customport` listener config from `cluster.hocon`: `bind = 0.0.0.0:18756`, `verify = verify_peer`, `fail_if_no_peer_cert = false`, TLS 1.2/1.3
- **Configure ACL** (`/etc/emqx/acl.conf`): copy verbatim from source. The `{allow, all}` final rule is the current production config — DO NOT change to `{deny, all}` without testing, as there are no per-topic allowlist rules configured; changing it would break all hub connections immediately. ACL hardening should be a separate Phase 7 security task.
- **QoS compatibility — reproduce QoS 0 (do NOT change to QoS 2):** The official firmware spec mandates QoS 2 (exactly-once) for all commands, but the current backend MQTT subscriber subscribes and publishes at QoS 0 (fire-and-forget). This is a known spec divergence and existing tech debt. The migration must reproduce QoS 0 to match the backend. Do not "fix" EMQX to QoS 2 during migration — that would require simultaneous backend code changes and would break the data pipeline. Log a post-migration tech debt item to align QoS level across firmware spec, broker config, and backend.
- **Do not yet point any devices to this broker.** It should be idle but ready to accept connections.
- Validate using the **`mqtt-prototype` simulator** (see [Trial device capability](#trial-device-capability)) before any DNS change.

**Security group — MUST include all of the following (copy exactly from source):**
- Port **1883** TCP: prod VPC CIDR, API ALB SG, smoke-api SG, all KORE/ONO SIM CIDRs (~20 blocks)
- Port **18756** TCP: all KORE/ONO SIM CIDRs + Numens testing IP (`223.94.44.49/32`)
- Port **22** TCP: OpenVPN VPC SG only (NOT from the internet)
- Management port **18083**: OpenVPN VPC SG only (NOT from the internet)
- **Do NOT recreate** the `116.73.36.32/32` ("lokendra") all-traffic rule — this is a personal developer IP with full access and is a security risk. Use OpenVPN for all developer access.

### 3.2 — Kafka broker

**Confirmed version:** Kafka 3.0.0 (Scala 2.13), ZooKeeper mode, single broker.

Steps:
1. Launch `t3.medium` EC2 in private subnet. Attach a dedicated EBS volume (20 GB minimum) mounted at `/data/kafka-logs`.
2. Install Kafka 3.0.0 matching the source. Install ZooKeeper 3.6.3 (or upgrade to KRaft — see note below).
3. Configure `server.properties`:
   - `listeners=PLAINTEXT://0.0.0.0:9092` (bind all interfaces, not a specific hostname)
   - `advertised.listeners=PLAINTEXT://<private-ip>:9092` (use private IP, not public DNS)
   - `log.dirs=/data/kafka-logs` (**not `/tmp`** — fix the source broker's bug)
   - `log.retention.hours=168` (7 days, matching source)
   - `offsets.topic.replication.factor=1` (single broker)
4. Create topic: `kafka-topics.sh --create --topic logs-production --partitions 1 --replication-factor 1 --bootstrap-server localhost:9092`
5. **Do not import data** — consumers reconnect and resume from `latest`. No Kafka data migration needed.
6. Security group: allow port 9092 from API server SG only.

**KRaft note:** Kafka 3.0+ supports ZooKeeper-less (KRaft) mode. Recommended for new deployments — eliminates ZooKeeper, simpler operation. If using KRaft, skip ZooKeeper install and use `kafka-storage.sh format` to initialise the cluster.

**Source broker bug to not carry forward:** `log.dirs=/tmp/kafka-logs` on source will wipe all Kafka data on any reboot. This is harmless for migration (Kafka is stateless for our purposes) but should be noted as a risk on the source until cutover.

### 3.3 — API server cluster

- Launch 5× `t3.medium` EC2 instances in private subnets.
- Use the existing `appspec.yml` and CodeDeploy configuration adapted for the new account (new deployment bucket, new CodeDeploy service role ARN).
- Configure environment variables to point at the new RDS endpoints, new Redis endpoint, new MQTT broker, new Kafka broker.
- Create a new `SmokeAPI-LB` ALB in the new account pointed at these instances.
- Test API health endpoints before proceeding.

### 3.4 — SSO provider

- Launch `t3.small` EC2 instance, deploy via `code/sso-provider/appspec.yml` adapted for new account.
- Create a new `sso-api-lb` ALB.
- Test OIDC endpoints.

### 3.5 — Keychain service

- Launch `t3.small` EC2 instance, deploy keychain app.
- Create `keychain-production-ALB`.
- **⚠️ Critical:** All environments (`keychain-development.*`, `keychain-qa.*`, `keychain-sandbox.*`, `keychain.*`) currently share the **same ALB** in the source account. The new account keychain service must be live and tested before the `keychain.sensorglobal.com` DNS record is updated. Once DNS is updated, all environments will switch to the new keychain simultaneously — plan non-prod environment readiness accordingly.

### 3.6 — Odoo

- Launch `t3.large` EC2 instance.
- Point at the restored `odoo-production` PostgreSQL RDS endpoint in the new account.
- Validate Odoo admin interface, CRM workflows, and outbound notifications before cutover.

### 3.7 — Supporting services

- **OpenVPN:** `t2.micro`, new certificates issued for new account. Set up first — required for SSH access to private instances (EMQX, Kafka, etc.) during build-out.
- **Cron server:** `t2.micro` instance (`smoke-prod-api-cron-server`). Deploy the cron jobs from the same codebase. Update cron job environment to point at new DB, Redis, and API endpoints. **Note:** understand the cron job schedule and confirm no jobs fire during the maintenance window.
- **WordPress (`wordpress-prod`):** `t3.medium`, behind ALB, no public IP in source account. Determine what DNS record serves it (check for `www.sensorglobal.com` or `sensorglobal.com` — separate from Odoo). May require a separate ALB or share the Odoo ALB.
- **WordPress SensorInsure (`wordpress-sensor-insure`):** `t2.medium`, has public IP `54.252.123.97` in source account. Determine if DNS record points directly to IP or via ALB.
- **Power BI Gateway:** `t3.large`, reconnect to Power BI Service with new gateway credentials.

### 3.8 — Frontend (Angular portals)

> **Finding (INV-04):** All 6 portals are in ONE S3 bucket (`smokealarmprod-admin`) served by ONE CloudFront distribution. New account needs one new S3 bucket and one new CloudFront distribution. Do not create 6 separate buckets — replicate the single-bucket architecture.

- Create ONE new S3 bucket for all portals in the new account.
- Create ONE new CloudFront distribution pointing at this bucket.
- Issue a new ACM wildcard certificate in `us-east-1` for `*.sensorglobal.com` via DNS validation (current cert is EXPIRED and INELIGIBLE for renewal — see prior evaluation).
- Trigger `Smoke-production-admin` CodePipeline (recreated in new account) to build and deploy Angular SPA.
- Validate each portal loads correctly against the new API base URL before DNS flip.

---

## Phase 4: MQTT Broker Cutover

> **Summary:** The single highest-risk moment in the migration. All physical field devices (smoke alarm hubs) will disconnect and reconnect. The key safety gate is the `mqtt-prototype` simulator — run it against the new broker in advance to confirm hub credentials, TLS handshake, and all command types work before touching DNS. Pre-reduce DNS TTL to 60–300 seconds 48 hours before the window.
> **⚠️ QoS:** The new EMQX must reproduce the current QoS 0 subscribe/publish model. Do NOT set QoS 2 (firmware spec value) — backend uses QoS 0 and must match. See Section 3.1 note.
> **Rough effort:** 48 hours pre-work (TTL reduction) + 1 maintenance window (2–4 hours execution + monitoring). Blocked until: Phase 2 (data in new account), Phase 3.1 (new EMQX running), INV-01 (hub credential model confirmed).

**Depends on:** Phase 2 (data in new account), Phase 3.1 (new EMQX running and validated). **INV-01 complete** — hub credential model confirmed (built-in DB, `emqx_ctl data export`/`import`).

**This is the highest-risk phase.** Devices will disconnect from the old broker and reconnect to the new one. The reconnection is automatic if:
- DNS TTL for `mqtt.sensorglobal.com` is pre-reduced (to 60–300 seconds)
- Hub firmware retries on connection failure
- Hub broker credentials are valid on the new broker

**Pre-cutover gate — simulator validation (mandatory):**

Before touching DNS, run the `mqtt-prototype` simulator against the new EMQX broker to confirm it is functioning correctly. See [Trial device capability](#trial-device-capability) for setup. The simulation must succeed for all of these command types: `VERIFY`, `ADD`, `REMOVE`, `UPDATE`, `TEST`, `BATTERY`, `ALARMS`, `FAULT`, and `LOW_BATTERY`. Do not proceed to the DNS change until this gate is passed.

> **Note on FAULT and LOW_BATTERY:** The firmware spec (reviewed 2026-04-12) confirms these are real device-emitted commands. The current backend silently discards them (known implementation gaps — see `docs/investigations/2026-04-09-alarm-event-traces.md`). However, the broker must still receive and route them through to Kafka. Include them in the simulator gate to confirm the broker pipeline handles them end-to-end, even if the app layer does not yet act on them.

**Cutover sequence:**
1. Pre-reduce DNS TTL for `mqtt.sensorglobal.com` to 60 seconds at least 48 hours before cutover.
2. On the new EMQX broker: confirm all hub credentials are loaded, ACLs are correct, TLS certificate is valid.
3. At cutover moment: update the `mqtt.sensorglobal.com` A record from the old IP to the new IP.
4. Monitor: new EMQX connected clients rising, old EMQX connected clients falling.
5. Keep the old EMQX broker running for at least 24 hours after cutover to catch any slow-to-reconnect hubs.
6. Monitor backend API for MQTT event processing (alarm logs, battery updates) to confirm the data flow is working end-to-end on the new account.

**Fallback:** Restore old DNS A record to roll back. All hubs reconnect to old broker automatically.

---

## Phase 5: Edge, DNS, and Certificate Cutover

> **Summary:** Flip all user-facing traffic to the new account — API, web portals, SSO, CloudFront. This requires a valid ACM certificate in both `us-east-1` (for CloudFront) and `ap-southeast-2` (for ALBs) before the DNS changes. The expired current wildcard cert in the source account is an active risk; issuing a fresh cert early (even in Phase 1) is recommended. Pre-reduce DNS TTLs 48 hours before the window.
> **Rough effort:** 48 hours pre-work (TTL reduction, cert pre-validation) + 1 maintenance window (2–4 hours). Blocked until: Phase 3 compute running, Phase 4 MQTT stable, new ACM cert issued and validated.

**Depends on:** Phase 3 (all compute running in new account), Phase 4 (MQTT cutover confirmed stable), a valid TLS certificate in the new account's ACM.

**Goal:** Flip production DNS so all user-facing traffic goes to the new account's ALBs and CloudFront distributions.

### 5.1 — Certificate pre-work
- Issue new ACM cert in `us-east-1` (for CloudFront) and `ap-southeast-2` (for ALBs) via DNS validation against `sensorglobal.com`.
- DNS validation must succeed before any CloudFront distribution or ALB HTTPS listener can use the cert.
- This step requires temporary Route 53 CNAME records to be added to `sensorglobal.com` for validation, which happens in the source account's Route 53 unless DNS has already moved.

### 5.2 — Pre-reduce DNS TTLs
- At least 48 hours before cutover, reduce TTL on all production-critical A records (api, auth, keychain, web, admin, agent, trader, owner, activation, reschedule, monitoring) to 60 seconds.

### 5.3 — ALB and CloudFront record flip sequence
Recommended order (lowest to highest impact if something goes wrong):
1. `web.sensorglobal.com` → new `sso-api-lb`
2. `auth.sensorglobal.com` → new SSO ALB
3. `keychain.sensorglobal.com` → new keychain ALB
4. `api.sensorglobal.com` → new `SmokeAPI-LB`
5. CloudFront aliases (admin, agent, trader, owner, activation, reschedule) → new distributions
6. `sensorglobal.com` (root A) → new Odoo/WordPress production host

### 5.4 — WAF
- Create WAF ACLs in the new account matching `Sensor_Global_Prod_web_ACL` and `Sensor_Global_Prod_key-chain_web_ACL` rules.
- Attach to the new ALBs and CloudFront distributions before traffic flip.

### 5.5 — Route 53 zone migration (decision-dependent, see INV-06)
- Option A: Keep `sensorglobal.com` hosted zone in the source account; only update records to point to new account's IPs/ALBs/CloudFront. Zone stays in source account indefinitely.
- Option B: Create the zone in the target account, pre-populate all records, then update NS records at the registrar to point to the new hosted zone. This is higher risk but cleaner long-term.
- Option A is recommended for the initial cutover to reduce blast radius.

---

## Phase 6: Third-Party Re-Onboarding and Full Validation

> **Summary:** Re-register all external integrations from the new account identity (KORE, PropertyMe, Twilio, SendGrid, Xero, etc.) and run end-to-end validation against the new environment using real-workflow smoke tests. **KORE is the critical path wildcard** — it requires re-registering the JV entity and waiting for KORE to issue new SIM API credentials. Start the KORE engagement in Phase 0 / Week 1, not here. The remaining integrations (PropertyMe webhook, SendGrid domain auth, Twilio number) should each take 1–4 hours.
> **Rough effort:** 1–2 weeks total. KORE lead time is unknown (could be 2–4 weeks if not started early). All other integrations can be done in a single day. Blocked until: Phase 5 DNS cutover, KORE re-registration (started in Phase 0).

**Depends on:** Phase 5 (all DNS pointing at new account).

**Goal:** Re-register and revalidate all third-party integrations from the new account's identity, and confirm end-to-end operational flows work.

### 6.1 — KORE SIM management
- Register the new account with KORE and obtain a new API key pair.
- Update the application environment config (`tbl_kore_api_keys` in the DB and/or the secrets in the new account's Secrets Manager).
- Validate SIM status checks are returning correctly for a sample of hub SIM IDs.

### 6.2 — PropertyMe
- Re-register the PropertyMe webhook callback URLs with the new `api.sensorglobal.com` origin.
- Validate OAuth token flows work from the new API host.
- Confirm property synchronization events are flowing.

### 6.3 — Twilio and SMS
- Update Twilio account SID and auth token in the new account's secrets.
- Confirm outbound SMS delivery for alarm notifications.

### 6.4 — SendGrid
- Update SendGrid API key(s) in the new account's secrets (`SENDGRID_API_KEY` and `SENDGRID_API_KEY_FOR_READING`).
- Confirm sender domain DNS records for `sensorglobal.com` are preserved: SPF, DKIM (two SendGrid selector sets: `u23692234` and `u24377603`).
- **DMARC hardening:** The source account has `DMARC p=none` (not enforcing). Set `p=quarantine` in the new account's DNS as part of email setup. Update the DMARC TXT record in `sensorglobal.com` to `v=DMARC1; p=quarantine; rua=mailto:<abuse-inbox>`.
- Confirm transactional email delivery for alarm notifications.

### 6.5 — New Relic
- **Decision confirmed:** New Relic is being dropped. No re-onboarding needed.
- Monitoring is handled by CloudWatch in the new account (see Phase 1.5).
- Remove `NEW_RELIC_LICENSE_KEY` from the new account's `sensor-prod` secret.

### 6.6 — Sentry (error tracking)
- `SENTRY_DNS_URL` (Sentry DSN) is present in `sensor-prod`. Sentry is in use for application error tracking.
- Create a new Sentry project for the new environment (or reuse the existing project — confirm with the dev team whether Sentry is tied to the old AWS account in any way).
- Update `SENTRY_DNS_URL` in the new account's `sensor-prod` secret.

### 6.7 — LaunchDarkly (feature flags)
- `LAUNCH_DARKLY_SDK_KEY` is present in `sensor-prod`. LaunchDarkly is in use for feature flag management.
- LaunchDarkly is account-agnostic (it is a SaaS product). The SDK key does not change with AWS account migration.
- Verify the key is still valid and the correct environment is targeted (prod vs staging) in the new account's secret.
- **🚨 `CommunicationsQueue` flag — preserve during migration:** The Feature Flags doc (reviewed 2026-04-12) confirms a LaunchDarkly flag named `CommunicationsQueue` ("Enables the overnight communications queue") is ON in all environments including production. Live DB investigation (2026-04-12) confirmed `tbl_communications_queue` is an **email-only** batch queue — it is NOT the SMS pathway. The P1 SMS drop root cause is a separate backend code change that broke the `tbl_alarm_alerts` insertion pipeline (see `docs/investigations/2026-04-12-mysql-sms-root-cause.md`). Regardless, **do not change the state of `CommunicationsQueue` during migration** — it governs email batch delivery and its state should be preserved. Verify its ON state in the new account post-cutover.

### 6.8 — PropertyMe and PropertyTree
- **PropertyMe:** Re-register OAuth app callback URL with the new `api.sensorglobal.com` origin. Update `PROPERTYME_API_CLIENT_ID` / `PROPERTYME_API_CLIENT_SECRET` / `PROPERTYME_TOKEN_URL` in the new secret if re-registration generates new credentials. Validate OAuth token flows and property synchronization events.
- **PropertyTree:** Separate integration from PropertyMe. Update `PROPERTYTREE_APPLICATION_KEY` and `PROPERTYTREE_SUBSCRIPTION_KEY` in the new secret. Verify if re-registration is required for the new JV entity.

### 6.9 — Console Cloud, NEXU, and Corpsure (insurance integrations)
- **Console Cloud:** Partner code `SENSOR` is set. Update `CONSOLE_CLOUD_API_CLIENT_ID` / `CONSOLE_CLOUD_API_SECRET` in the new secret. Verify with Console Cloud whether the JV entity needs a new partner registration.
- **NEXU:** `NEXU_API_TOKEN` present. Verify if token is account-neutral or needs re-issuance for the new entity.
- **Corpsure:** Credentials (`CORPSURE_LOGIN_USERNAME/PASSWORD`) and Google Cloud Functions endpoints present. Corpsure is an external insurance partner. Verify if the new JV entity needs a new Corpsure account or if existing credentials carry over.

### 6.10 — Google (reCAPTCHA, Maps, Firebase)
- **reCAPTCHA Enterprise:** `RECAPTCHA_URLSECRET_KEY`, `RECAPTCHA_API_KEY`, and `RECAPTCHA_PROJECTID` present. The reCAPTCHA project ID is Google Cloud project-specific. If the new JV company creates a new GCP project, create a new reCAPTCHA site key and update these values.
- **Maps/Places:** `GOOGLE_MAPS_PLACES_API_KEY` — check if usage is billed to a specific GCP project. Update if a new GCP project is used.
- **Firebase:** `sensor-production` Firebase project is referenced in the secret. Firebase push notifications for hub alerts may depend on this project. Verify if Firebase is in use for mobile/web push and whether the project needs to be migrated to the new GCP/JV entity.

### 6.11 — Zabbix monitoring
- `zabbix_monitoring` IAM user exists — Zabbix is (or was) in use for infrastructure monitoring alongside New Relic.
- **Confirm with ops team:** Is Zabbix actively used? If yes, recreate the `zabbix_monitoring` IAM user in the new account with the same monitoring permissions and reconfigure the Zabbix server to point at the new account's resources.
- If not in active use, do not migrate.

### 6.12 — Odoo SaaS / Odoo integrations
- Confirm the self-hosted Odoo instance in the new account can reach any external Odoo SaaS endpoints or APIs it was using.
- Validate the PropertyMe → Odoo synchronization flow.

### 6.13 — OpenVPN / operational access
- Issue new OpenVPN certificates.
- Confirm the operations team can connect to all private-subnet resources.

### 6.14 — End-to-end validation

The following validation tools are available for testing the new environment without needing physical hardware. See [Trial device capability](#trial-device-capability) for full setup details.

**Validation checklist:**
- Run `mqtt-prototype` simulator against the new EMQX broker — trigger VERIFY, ADD, TEST, BATTERY, ALARMS sequences and confirm responses are received.
- Use the `Test Pack` API endpoint (`POST /test-pack/setup`) to create a fully populated test agency with properties, users, jobs, contractors, and invoices. Confirm end-to-end job flow.
- Confirm alarm event appears in MySQL alarm logs and MongoDB, outbound notifications sent via Twilio/SendGrid, and event visible in Angular portals.
- Run pre/post migration counts on key tables and confirm they match source.
- Confirm a job can be created, assigned, and completed in the new account.

---

## Phase 7: Source Account Decommission

> **Summary:** Controlled wind-down of the old account after confirming the new account is stable. Maintain a minimum 2-week soak period with source resources kept intact (cold standby) before terminating anything. Non-prod environments (qa, dev, sandbox, uat) should be archived/snapshotted and terminated — they are not being migrated initially. Odoo is migrated only if it passes user validation in the new account first.
> **Rough effort:** 2–3 weeks soak period + 1 week cleanup. Blocked until: Phase 6 validation complete, 2 weeks of confirmed stable operation.

**Depends on:** Phase 6 validation complete, at least 2 weeks of stable operation on the new account.

**Goal:** Cleanly shut down the source account, archive what is needed, and exit.

### 7.1 — Non-production environments (qa, dev, sandbox, uat)
- Confirm with the JV stakeholders whether any non-prod environments are needed in the new account.
- If yes, migrate them following the same pattern (simplified — no MQTT cutover needed for non-prod; a maintenance window snapshot+restore is sufficient).
- If no, take final RDS snapshots and S3 archive dumps of any valuable QA/dev data, then terminate.

### 7.2 — Stop production services in source account (after validation period)
- Terminate EC2 instances (keep snapshots for 90 days).
- Take final RDS snapshots and archive to long-term S3 (Glacier tier).
- Delete CloudFront distributions, ALBs, and Route 53 records (or keep Route 53 records pointing to new account indefinitely — see INV-06).
- Cancel or transfer marketplace subscriptions (OpenVPN, Vanta if applicable).
- Remove VPN, networking, and IAM resources.

### 7.3 — Final DNS and domain actions

**Route 53 Registrar — domain transfer (confirmed: all 14 domains are in Route 53 Registrar):**
1. Confirm AutoRenew payment method is still valid on the source account. **Do not close the source account while any domain AutoRenew is pending.** Several domains renew mid-2026: `sensorglobal.au` / `.net` (June), `.co.uk` / `.com.au` (July), `sensorinsure.com` / `.com.au` (September).
2. Disable TransferLock on `sensorglobal.com` and `sensoriq.co.uk` (currently locked) before initiating domain transfer.
3. Initiate domain transfer from source account to new account via Route 53 Registrar console.
4. After domain transfers complete, migrate Route 53 hosted zones (create zone in new account, populate all records, then update NS records at the registrar).

**Zone migration alternative:** Keep hosted zone in source account permanently (source account becomes a DNS-only billing account). This avoids zone migration risk but means the source account can never be fully closed.

### 7.4 — Account closure
- Confirm zero active resources and zero charges.
- Contact AWS to close the source account or move it to a suspended state within its organization.

---

## Required Deeper Investigation Summary

| ID | Title | Blocks Phase | Priority | Status |
|---|---|---|---|---|
| INV-01 | EMQX broker config and hub credential model | Phase 4 (MQTT cutover) | **Critical** | ✅ Complete (SSH 2026-04-10) |
| INV-02 | MongoDB inventory and volume | Phase 2.2 (data migration) | **Critical** | ✅ Complete |
| INV-03 | ElastiCache Redis inventory and TTL patterns | Phase 2.4 (Redis migration) | High | ✅ Complete |
| INV-04 | S3 bucket inventory and CloudFront origins | Phase 3.8 (frontend rebuild) | High | ✅ Complete |
| INV-05 | IAM, secrets, and credentials audit | Phases 1.4, 3, 6 | **Critical** | ✅ Complete |
| INV-06 | DNS and registrar strategy decision | Phase 5 (DNS cutover) | **Critical** | ✅ Complete |
| INV-07 | Deployment automation and CI/CD inventory | Phase 3 (compute rebuild) | High | ✅ Complete |
| INV-08 | Active user and device headcount | Validation criteria | Medium | ✅ Complete (SSH for device count) |

---

## Trial Device Capability

The repository contains two tools that make it possible to validate the new environment's MQTT broker and application tier **without needing a physical hub device**. These should be used as mandatory gates in Phase 3 (pre-DNS) and Phase 6 (end-to-end validation).

### mqtt-prototype — Hub controller simulator

**Location:** `code/mqtt-prototype/` (.NET 7 C# console app)

This tool simulates one or more physical SensorGlobal hub controllers. It connects to any MQTT broker and drives the full command/response protocol.

**Configuration** (`code/mqtt-prototype/config.json`):
```json
{
  "MQTT_HOST": "<new-emqx-host>",
  "MQTT_PORT": 8883,
  "MQTT_USERNAME": "<broker-username>",
  "MQTT_PASSWORD": "<broker-password>",
  "USE_TLS": true,
  "ACTIVATION_URL": "https://api.sensorglobal.com/activation",
  "SENSOR_API_BASE_URL": "https://api.sensorglobal.com"
}
```

**Hub serial numbers** are listed in `code/mqtt-prototype/controllers.json`.

**Topics it uses:**
- Subscribes to: `sg/sas/cmd/{serialNumber}` (commands from server to hub)
- Publishes to: `sg/sas/resp/{serialNumber}` (responses from hub to server)

**Commands it handles:** `VERIFY`, `ADD`, `REMOVE`, `UPDATE`, `TEST`, `BATTERY`, `ALARMS`

**How to use for new-environment validation:**
1. Point `MQTT_HOST` at the new EMQX broker's IP or DNS name.
2. Set `MQTT_USERNAME` / `MQTT_PASSWORD` to credentials loaded in the new broker (from INV-01 output).
3. Run `dotnet run` in the project directory.
4. Send commands to the simulator hubs via the API and observe that response payloads arrive correctly on the server side.
5. Trigger an ALARMS event and confirm it flows through to MySQL/MongoDB and triggers a notification.

### Test Pack — API-driven test agency creation

**Location:** `code/sensor-alarm-backend/src/controllers/users.testpack.controller.ts`

This backend feature (SENS-5415) creates a fully populated test agency in the running application — including properties, users (agents, contractors, tenants), jobs, invoices, and costings — via a single API call. It is intended for end-to-end validation of the full application flow without requiring real customer data.

**How to use:** Send a `POST` request to the test pack endpoint on the new API server. Requires the `DUMMY_ABN` environment variable to be set. Review the controller for the exact endpoint path and request shape.

### sensor-cli — Database test data seeder

**Location:** `code/sensor-cli/`

Node.js CLI tool with `mock_data.sql` for seeding test users and data directly into MySQL. Useful for populating the new database with baseline test data before running application-layer tests.

---


## Open Questions (Requiring Human Decision)

~~1. **Target account region**~~ ✅ Confirmed: `ap-southeast-2`
~~2. **Decommission deadline**~~ ✅ Confirmed: ASAP — phases must run in parallel
~~3. **Domain ownership**~~ ✅ Confirmed: keep `sensorglobal.com` (JV inherits)
~~4. **Non-production environments**~~ ✅ Confirmed: prod-only migration initially
~~5. **Odoo deployment model**~~ ✅ Confirmed: keep self-hosted
~~6. **KORE SIM contract**~~ ✅ Confirmed: current company holds contract — re-registration required
~~7. **New Relic account**~~ ✅ Confirmed: drop New Relic entirely — use CloudWatch
~~8. **Acceptable downtime window**~~ ✅ Confirmed: maintenance window acceptable (hours, off-peak)

**New questions emerged from investigations (unresolved):**

9. **Does the JV company retain Appinventiv as the development team?** Six Appinventiv IAM users exist (`deepak`, `gurpreet`, `jeetendra`, `lokendra`, `rudresh` plus `Appinventiv_S3`). Their access needs to be recreated or terminated in the new account. The individual developer `116.73.36.32/32` with full-traffic EMQX access must not be carried forward.

10. **What DNS record serves `wordpress-prod`?** It has no public IP (behind ALB). Need to identify its ALB and the DNS record pointing to it. Affects Phase 3.7 and Phase 5.3 sequencing.

11. ~~**What is EMQX port 18756?**~~ **✅ Resolved (2026-04-10 SSH):** Port 18756 is the primary hub MQTT-over-TLS listener. 16 of 21 connected clients use it. The `*.sensorglobal.com` cert on this port expired 2026-04-08; hub firmware ignores cert expiry. New broker must reproduce this listener with a valid cert.

12. **Is Zabbix still actively used?** The `zabbix_monitoring` IAM user exists but Zabbix was not visible in the EC2 inventory (may run on the Odoo server or outside AWS). If active, it must be reconfigured for the new account. If not, do not migrate.

13. **Does the JV entity need a new MongoDB Atlas organization?** Atlas billing is separate from AWS. The current cluster is under `sensor-prod.vr48f.mongodb.net`. Access to atlas.mongodb.com is needed to confirm cluster tier, billing, and whether a new Atlas org is required for the JV entity.

14. **Is Google Firebase used for push notifications?** `sensor-production` Firebase project is referenced in the secret. If Firebase Cloud Messaging (FCM) is used for mobile app alarm alerts, it must be migrated to the JV entity's GCP project before cutover.

---

## Decisions Confirmed (2026-04-10)

| # | Answer | Plan impact |
|---|---|---|
| Region | `ap-southeast-2` | No change |
| Deadline | **ASAP** | Phases 1, 2, and 3 must run **in parallel**. Do not wait for data migration before starting compute rebuild. |
| Domain | **Keep `sensorglobal.com`** | DNS cutover plan unchanged. Route 53 zones must be accessible to new account for validation and eventual transfer. |
| Non-prod | **Prod-only initially** | QA/dev/sandbox/UAT can be archived and terminated. Non-prod migration is out of scope for this engagement. |
| Odoo | **Keep self-hosted** | Phase 3.6 and Phase 2.3 as planned. |
| KORE | **Current company holds contract** | **Highest-priority pre-work**: KORE vendor engagement must start immediately, before Phase 1. Re-registration for a new entity can take weeks. Do not treat KORE as a Phase 6 item. |
| Monitoring | **CloudWatch only** (drop New Relic) | No Metric Stream, no Firehose, no New Relic agent. CloudWatch agent on all EC2 hosts. Log retention policy set at creation time — see Phase 1.5. |
| Downtime | **Maintenance window acceptable** | MySQL and MongoDB can use snapshot+restore in a planned window rather than live replication. Redis cold-start is fine. Simplifies Phase 2 significantly. |

---

## Risks by Phase

| Phase | Highest risk | Mitigation |
|---|---|---|
| 0 (Investigations) | ~~INV-01 reveals hub credentials are per-device and cannot be remotely updated~~ **✅ Resolved:** Credentials are in EMQX built-in DB, exportable via `emqx_ctl data export` | Use `data export`/`data import` between source and new broker |
| 1 (Account setup) | Log groups created without retention — accumulate storage costs from day one | Set retention on every log group at creation; never allow `None`; see Phase 1.5 retention policy table |
| 1 (Account setup) | IAM hardcoded access keys in `sensor-prod` secret (`AWS_ACCESS_KEY`/`AWS_SECRET_KEY`) carried to new account | Replace with EC2 instance roles; never use long-term IAM user keys for instance auth in new account |
| 2 (Data migration) | Odoo 100 GB backup corrupted or restore takes longer than window | Test restore in a staging/dev environment first; allocate extra window time for Odoo |
| 2 (Data migration) | MongoDB Atlas cluster size or tier unknown — may be undersized for a new Atlas project | Get Atlas console access to confirm tier before provisioning the new cluster |
| 3 (Compute rebuild) | Environment variable mis-configuration causes API to connect to source DB or broker | Use strict environment validation at startup; run a full smoke test before any traffic is shifted |
| 3 (Compute rebuild) | EMQX SG missing one or more KORE/ONO SIM CIDRs — some hub SIMs cannot reach new broker | Copy the full SG rule list from `docs/investigations/2026-04-10-inv-results.md` INV-01 exactly |
| 3 (Compute rebuild) | Missing cron server or WordPress from rebuild scope causes operational gaps | Explicitly include `smoke-prod-api-cron-server` and `wordpress-prod` in Phase 3.7 checklist |
| 4 (MQTT cutover) | Hub firmware does not retry on connection failure; devices go permanently offline | Test with a development hub before cutover; keep old broker running for 24+ hours as fallback |
| 4 (MQTT cutover) | **🚨 TLS cert on port 18756 expired 2026-04-08** — renew on source broker immediately; new broker must start with valid cert | Provision a new `*.sensorglobal.com` cert via Let's Encrypt or ACM before launching new EMQX; hub firmware ignores expiry but is still a security risk |
| 5 (DNS cutover) | ACM cert not yet validated; CloudFront rejects HTTPS traffic | Issue cert days before cutover and confirm validation status; use HTTP health checks on new distributions before DNS flip |
| 5 (DNS cutover) | Keychain DNS update breaks all environments simultaneously (all share one ALB) | Ensure non-prod keychain is also running in new account before `keychain.*` DNS is updated |
| 6 (Third-party) | **KORE re-registration takes weeks** — SIM status updates broken post-cutover | **Start KORE engagement immediately** (before Phase 1); contract is in current company's name and re-registration for a new entity is vendor-controlled |
| 6 (Third-party) | `CommunicationsQueue` LaunchDarkly flag accidentally toggled — overnight **email** batch delivery disrupted | Verify `CommunicationsQueue` is ON in new account post-cutover before go-live; do not change it without app team sign-off. Note: queue is email-only — SMS pipeline is separate (see `2026-04-12-mysql-sms-root-cause.md`) |
| 6 (Third-party) | Google Firebase project not migrated — push notifications stop working | Verify Firebase usage (mobile push for alarm alerts) before cutover; migrate project to new GCP/JV entity |
| 7 (Decommission) | Domain AutoRenew payment fails because source account payment method is removed before domains transfer | Confirm payment method is valid before closing account; several domains expire mid-2026 |
| 7 (Decommission) | Data accidentally deleted before validation period is complete | Enforce a minimum 2-week soak period; keep final snapshots for 90 days |
