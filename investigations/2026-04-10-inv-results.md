# INV-01 through INV-08: Investigation Results

**Date:** 2026-04-10  
**Account:** `747293622182`, `ap-southeast-2`  
**Method:** Read-only AWS CLI queries + local codebase analysis + SSH to `Emqx-prod` via OpenVPN  
**Status:** INV-01 ✅ complete (SSH obtained 2026-04-10); INV-02 through INV-08 complete

> **Security hygiene update (2026-04-27):** Earlier revisions of this note included literal MQTT and MongoDB credential values copied from `sensor-prod`. Those values have been redacted here. Treat the previously exposed values as leaked if this repository was shared outside the approved investigation audience, and rotate them through the normal production change process.

---

## INV-01: EMQX Broker Configuration and Hub Credential Model ✅

> SSH obtained to `Emqx-prod` (`i-0b381a2bfdf65a329`) via OpenVPN on 2026-04-10. All items resolved.

### EMQX version and topology

- **Version:** EMQX **5.8.7** (recent, production-grade)
- **Topology:** **Single node** — `emqx@127.0.0.1`, `discovery_strategy = manual` — no clustering
- **Connected clients at time of investigation:** 21 total (5 on port 1883, 16 on port 18756, 0 on others)

### Listeners (from `cluster.hocon` + `emqx_ctl listeners`)

| Listener | Port | Protocol | Current connections | Notes |
|---|---|---|---|---|
| `tcp:default` | **1883** | Plain MQTT (TCP) | 5 | API servers (internal VPC) + some hubs |
| `ssl:customport` | **18756** | **MQTT over TLS** | **16** | **Primary hub port** — custom TLS cert |
| `ssl:default` | 8883 | MQTT over TLS | 0 | Configured but not in use |
| `ws:default` | 8083 | MQTT over WebSocket | 0 | Enabled but not in use |
| `wss:default` | 8084 | MQTT over WebSocket+TLS | 0 | Enabled but not in use |

**Port 18756 = MQTT over TLS (custom port).** This is the primary hub connection port. Hubs from KORE/ONO SIM cards connect via TLS MQTT on port 18756 using a custom `*.sensorglobal.com` certificate. 16 of the 21 connected clients are on this port.

**Historical shutdown counts on port 18756:**
- `keepalive_timeout`: 26,767 — high (normal for cellular MQTT with unreliable connectivity)
- `ssl_upgrade_timeout`: 6,057 — many TLS handshake failures (cellular radio drops mid-connect)
- `discarded`: 11,290 — significant (duplicate connects, stale sessions)
- `protocol_error`: 4,084 — some firmware protocol issues
- `not_authorized`: 2 — only 2 auth failures in history — auth is stable

### 🚨 CRITICAL: TLS certificate for port 18756 is EXPIRED

```
subject: C=AU, ST=Victoria, L=Sandringham, O=SENSOR PTY LTD, CN=*.sensorglobal.com
notBefore: Apr  9 00:00:00 2025 GMT
notAfter:  Apr  8 23:59:59 2026 GMT   ← EXPIRED 2 DAYS AGO
```

The wildcard cert used by the TLS listener on port 18756 expired on **2026-04-08**. Hubs are still connecting (16 active) because hub firmware does **not validate TLS certificate expiry** (and possibly not the trust chain either — consistent with embedded firmware on cellular IoT devices). The server is configured with `verify = verify_peer` / `fail_if_no_peer_cert = false`, which means the server optionally requests a client cert but doesn't require one — effectively one-way TLS with no client cert validation.

**Action:** Renew the `*.sensorglobal.com` certificate immediately on the source broker, and ensure the new broker starts with a valid cert from day one.

### Authentication

- **Backend:** EMQX **built-in database** (Mnesia/RocksDB internal to EMQX)
- **Mechanism:** Password-based, `user_id_type = username`
- **Password storage:** `plain` (no hashing, `salt_position = disable`) — **plaintext passwords stored in EMQX internal DB**

This means **hub credentials are stored as username/plaintext-password pairs inside EMQX's internal database.** They are NOT in MySQL, Redis, or any external service. To migrate the hub credentials to the new broker, the built-in database must be exported from the source and imported into the new instance.

**Export method (EMQX 5.x) — executed 2026-04-10 22:05 UTC:**
```bash
sudo emqx_ctl data export
# Output: /var/lib/emqx/backup/emqx-export-2026-04-10-22-05-29.420.tar.gz
# Size: 10,185 bytes (compressed)
```

Export succeeded cleanly. The tarball includes: `cluster configuration`, `emqx_authn_mnesia` (hub credentials), `emqx_acl`, `emqx_app` (API keys), `emqx_admin` (dashboard users), retained messages, PSK and banned tables.

**The `emqx_authn_mnesia` table has exactly 1 entry** (confirmed via `emqx eval "ets:tab2list(emqx_authn_mnesia)."` on 2026-04-10):

```erlang
[{user_info,{'mqtt:global',<<"<redacted-shared-username>">>},
            <<"<redacted-shared-password>">>,<<>>,false}]
```

**This is the same credential already known from the `sensor-prod` Secrets Manager secret.** There is ONE shared credential used by ALL clients — both the 5 API server instances AND all 16 hub field devices. No per-device authentication exists.

Implication: the `data export` tarball is useful for restoring cluster config, ACL, and dashboard admin, but the credential migration is trivial — there is exactly one username/password pair to provision on the new broker, and it is already documented in the `sensor-prod` secret.

On the new broker:
```bash
sudo emqx_ctl data import /path/to/emqx-export-2026-04-10-22-05-29.420.tar.gz
```
The file must be transferred off the source instance before decommission. Copy to S3 or direct SCP.

**API-to-broker credential (from `sensor-prod` secret — unchanged):**
- `MQTT_USERNAME`: `<redacted>` (single shared UUID credential for all 5 API servers)
- `MQTT_PASSWORD`: `<redacted>`

### Authorization (ACL)

- **Source:** File-based — `/etc/emqx/acl.conf`
- **`no_match`:** `allow` — if no ACL rule matches, the action is **allowed**
- **ACL cache:** Enabled, 1-minute TTL, max 32 entries

**Effective ACL rules:**
```erlang
{allow, {username, {re, "^dashboard$"}}, subscribe, ["$SYS/#"]}.   % dashboard user only
{allow, {ipaddr, "127.0.0.1"}, all, ["$SYS/#", "#"]}.              % localhost full access
{deny, all, subscribe, ["$SYS/#", {eq, "#"}, {eq, "+/#"}]}.        % no wildcard sub to root/#
{allow, all}.                                                        % EVERYTHING ELSE ALLOWED
```

⚠️ **The ACL is effectively open.** The `{allow, all}` last rule (combined with `no_match = allow`) means any authenticated client can publish or subscribe to any topic. The `acl.conf` comment explicitly notes this should be `{deny, all}` in production, but it was never changed. No topic-level restrictions exist for hub or API clients.

**Copy this ACL file verbatim to the new broker.** (It is already permissive — no secret rules to preserve.)

### Security group (from prior AWS query)

| Port | Protocol | Sources |
|---|---|---|
| All | All | `116.73.36.32/32` (labeled "lokendra" — full access; **security risk — do not copy**) |
| **1883** | TCP | `10.0.0.0/16` (prod VPC), API ALB SG, smoke-api-sg, ~20 KORE/ONO SIM CIDRs |
| **18756** | TCP | ~20 KORE/ONO SIM CIDRs + `223.94.44.49/32` (Numens testing) |

Do NOT reproduce the `116.73.36.32/32` all-traffic rule. Replace developer access with OpenVPN.

### Impact on migration plan

| Item | Finding | Migration action |
|---|---|---|
| Hub credential model | **1 credential total** — same UUID pair used by all hub devices AND all API servers. No per-device auth. | No credential migration needed — credential is already in `sensor-prod` secret. Just configure it on new broker. |
| Hub client IDs | `<12-hex MAC/serial>C001<4-digit suffix>` — 12 of 16 devices have suffix `2406` (likely firmware 24.06) | Client IDs are set by device firmware — no migration action needed |
| API server client IDs | `mqttjs_<random>` — Node.js mqtt.js library, new random ID on each reconnect | Normal behavior; no migration concern |
| Connected hub IPs | `44.204.32.x`, `44.216.145.x`, `3.6x.x.x` — all AWS IP ranges (KORE SIM routing) | KORE/ONO SIM IPs — reproduce these CIDRs on new EMQX security group |
| Retained messages | Only `$SYS/` system topics — **no application retained state** | Nothing to migrate; clean cutover |
| Port 18756 | MQTT over TLS — **primary hub port** | Must be reproduced with same cert CN; new cert must be valid |
| TLS cert | **EXPIRED 2026-04-08** | Renew immediately; new broker must start with valid `*.sensorglobal.com` cert |
| ACL | Fully open (`{allow, all}`) | Copy acl.conf verbatim; optionally tighten to `{deny, all}` + per-topic rules |
| EMQX version | 5.8.7 | Install same version on new instance; `data export`/`import` is version-compatible |
| Single node | No cluster | Simple: one new `c5.xlarge` EC2, install EMQX 5.8.7, import data |
| Auth admin password | Changed from default | Use `data export` (bypasses API auth) for credential migration |

---

## INV-02: MongoDB — Host, Topology, and Data Volume

> **Major finding: MongoDB is MongoDB Atlas, not a self-hosted EC2 instance.**

### What we know

From the `sensor-prod` Secrets Manager secret:

```
MONGO_DB_URL:         mongodb+srv://<redacted>@sensor-prod.vr48f.mongodb.net/sensorproddb?readPreference=secondaryPreferred&w=majority
MONGO_DB_DATABASE:    sensorproddb
MONGO_DB_ARCHIVE_URL: mongodb://<redacted>@atlas-online-archive-67343af0eaeb997ac96cb7f9-vr48f.a.query.mongodb.net/sensorproddb?ssl=true&authSource=admin
KEEP_DATA_ALIVE_IN_CURRENT_MONGODB: 90  (days)
```

**Cluster:** `sensor-prod.vr48f.mongodb.net` — this is a **MongoDB Atlas** managed cluster.

**Atlas Online Archive:** The archive URL (`atlas-online-archive-...`) indicates Atlas Online Archive is configured, automatically archiving data older than 90 days from the live cluster to cheaper cold storage. The live cluster holds 90 days of active data; older data is archived.

**No MongoDB EC2 instance exists in the production VPC.** No `mongo*`-named instance was present in the prod EC2 inventory.

### What this means for migration

| Old assumption | Reality |
|---|---|
| Need to SSH into a MongoDB EC2 instance | No EC2 to SSH into |
| Plan a snapshot of EBS volume | Not applicable |
| Need to choose a MongoDB version to reinstall | Atlas version is managed |

**New approach — MongoDB migration:**

1. Create a new MongoDB Atlas cluster in the new JV entity's Atlas organization.
2. Use **Atlas Live Migration** (available in Atlas UI) to replicate from the source cluster to the new cluster without downtime, or use `mongodump` / `mongorestore` in the maintenance window.
3. The archive URL is account-bound — a new Atlas Online Archive configuration must be set up in the new Atlas project.
4. Update `MONGO_DB_URL` and `MONGO_DB_ARCHIVE_URL` in the new account's Secrets Manager secret after migration.
5. Set `KEEP_DATA_ALIVE_IN_CURRENT_MONGODB: 90` in the new secret as well.

**Action needed:** Get the Atlas organization access to see cluster size, Atlas tier, and project settings. The Atlas cluster may be paid for under a separate invoice not visible in AWS Cost Explorer.

---

## INV-03: Redis Key Patterns and Cold-Start Viability

> **Finding: Cold-start is safe. All application Redis keys use TTLs via `setEx`.**

### ElastiCache cluster facts

- Cluster: `smokeprodapiredis-001` (primary) + `smokeprodapiredis-002` (replica)
- Engine: Redis 6.2.6
- Node type: `cache.t3.medium` (each)
- Status: `available`
- Parameter group: `smokeapi` (custom)
- **Eviction policy:** `volatile-lru` — only keys with an expiry TTL are evicted under memory pressure. Keys without TTLs are permanent until explicitly deleted.
- Endpoint (from secret): `smokeprodapiredis.ngkvo9.ng.0001.apse2.cache.amazonaws.com:6379`

### Key namespaces (from code analysis)

All keys are set using `setDataInRedisWithExpireTime` which calls `pubClient.setEx(key, expirationTime, value)` — **all application keys have explicit TTLs**.

| Key pattern | Purpose | Notes |
|---|---|---|
| `controller_{id}_property_{id}_event_{CMD}` | Job deduplication for MQTT commands | Short TTL (minutes) |
| `single_controller_{id}_property_{id}_event_{CMD}` | Single-controller job dedup | Short TTL |
| `{serialNumber}_HUB_VERIFY` | Hub verification event deduplication | Short TTL |
| `{serialNumber}_{alarmSN}_{index}_{id}_{type}` | Alarm event deduplication | Short TTL |
| `at` / `nt` | Alarm test / notify alarm test constants | Likely short TTL |
| `propertyme_states_{state}` | PropertyMe OAuth 2.0 state parameter | Short TTL (OAuth flow timeout) |
| `{messageId}` | Mail deduplication | Short TTL |

### Cold-start recommendation: **SAFE TO COLD-START**

- All keys have TTLs via `setEx`
- `volatile-lru` policy will expire them naturally
- The worst case on cold-start: in-flight MQTT alarm deduplication keys are lost → alarm may be re-processed once. Acceptable during a maintenance window when MQTT is idle.
- User sessions are stored in MySQL `tbl_sessions`, NOT in Redis. No session data is lost.

**Migration approach:** Do NOT attempt a Redis snapshot migration. Just provision a fresh ElastiCache Redis cluster in the new account (same tier: `cache.t3.medium` primary + replica) and let the application re-populate it. Ensure the custom parameter group `smokeapi` is recreated with `maxmemory-policy = volatile-lru`.

---

## RDS Instance Inventory (AWS CLI — 2026-04-11)

### Production instances

| Instance ID | Engine | Version | Class | Storage | Used | MultiAZ | Encrypted |
|---|---|---|---|---|---|---|---|
| `sensor-prod` | MySQL | 8.0.44 | `db.t3.medium` | 20 GB gp2 | **11.7 GB (59%)** | ✅ Yes | ✅ Yes |
| `odoo-production` | PostgreSQL | 14.17 | `db.t3.medium` | 100 GB io1 (3000 IOPS) | **10.0 GB (10%)** | ⚠️ **No** | ✅ Yes |

### Non-production instances (not being migrated initially)

| Instance ID | Engine | Class | Storage | MultiAZ |
|---|---|---|---|---|
| `sensor-qa` | MySQL 8.0.44 | `db.t3.medium` | 50 GB gp3 | No |
| `sensor-sandbox` | MySQL 8.0.44 | `db.t3.medium` | 20 GB gp2 | Yes |
| `smokealaramuat` | MySQL 8.0.44 | `db.t3.medium` | 20 GB gp2 | Yes |
| `dev-db-temp-mysql-8` | MySQL 8.0.44 | `db.t3.micro` | 50 GB gp2 | No |
| `odoo-dev` | PostgreSQL 14.17 | `db.t3.medium` | 20 GB gp3 | No |
| `odoo-qa` | PostgreSQL 14.17 | `db.t3.medium` | 20 GB gp3 | No |

### sensor-prod (MySQL) — key settings

- **Parameter group:** `prod-mysql-8-parameter-group` (custom) — **must be recreated in new account**
  - `group_concat_max_len = 100000` (default is 1024 — application depends on this for aggregations)
  - `log_bin_trust_function_creators = 1` (required if stored procedures/functions use binary log)
- **Option group:** `default:mysql-8-0`
- **Active DB connections (recent):** 8 peak — very low
- **Backup window:** 14:00–15:00 UTC | **Maintenance window:** Sat 15:00–16:00 UTC
- **Backup retention:** 7 days
- **Deletion protection:** enabled
- **No read replicas**

### odoo-production (PostgreSQL) — key settings

- **Parameter group:** `default.postgres14` (no custom parameters)
- **Storage:** io1 with 3000 IOPS — **expensive and oversized** (only 10 GB used of 100 GB)
- **Active DB connections (recent):** 20 peak
- **MultiAZ:** **None** — Odoo has no standby; instance failure = Odoo outage
- **Backup retention:** 7 days | **Deletion protection:** enabled
- **Most recent manual snapshot:** `odoo-production-snapshot-aug-13` (2024-08-13) — stale

### Maintenance window estimate

| Database | Data size | Snapshot restore estimate | Notes |
|---|---|---|---|
| MySQL `sensor-prod` | 11.7 GB | 15–25 min | Multi-AZ snapshot adds ~10 min |
| PostgreSQL `odoo-production` | 10.0 GB | 15–25 min | io1 restore from 100 GB allocated (not data size) |
| **Combined** | ~22 GB | **~1 hour including buffer** | Can be parallelised |

### Findings and recommendations

1. **sensor-prod custom parameter group must be carried forward.** Both params are non-default and the application likely relies on them (`group_concat_max_len` at 100× default strongly suggests GROUP_CONCAT usage in queries).
2. **odoo-production io1 is oversized.** 100 GB io1 @ 3000 IOPS costs ~$330/month vs. gp3 100 GB @ 3000 IOPS at ~$115/month. Recommend switching to gp3 in the new account (same IOPS baseline, ~65% cheaper). Only 10 GB of data used.
3. **odoo-production has no Multi-AZ.** Odoo is presumably less critical than the main API, but worth noting. Decision for new account: keep Single-AZ (cheaper) or add Multi-AZ.
4. **All instances are encrypted at rest.** KMS key must be re-encrypted when sharing snapshots cross-account (share the KMS key or re-encrypt with the target account's KMS key).

---

## INV-04: S3 Buckets and CloudFront Origin Mapping

### Production CloudFront distributions

| Subdomain(s) served | CloudFront distribution | S3 origin bucket | Purpose |
|---|---|---|---|
| `admin.`, `agent.`, `trader.`, `owner.`, `activation.`, `reschedule.sensorglobal.com` | `d2obn6q80a5yi1.cloudfront.net` (E1UDNFPYDRUZOD) | `smokealarmprod-admin` | **ALL prod portals from ONE bucket** |
| `dxu6ibcxyrwaw.cloudfront.net` | `smokeapibucket` | App assets/documents/exports |

**`smokealarmprod-admin`** — 592 objects, ~24 MB. This is the compiled Angular SPA (all 6 portals deployed into a single S3 bucket, served by a single CloudFront distribution via path-based routing).

**`smokeapibucket`** — 31,743 objects, ~3.4 GB. This is the main application storage: user-uploaded documents, property photos, report exports, API config files, email assets, logo images, etc.

### S3 buckets — full production inventory

| Bucket | Purpose | Environment |
|---|---|---|
| `smokealarmprod-admin` | Angular SPA — all 6 portals | **Prod** |
| `smokeapibucket` | App documents/assets/storage | **Prod** |
| `smokealarm-agent` | (may be legacy/unused — portals now in smokealarmprod-admin) | Possibly legacy |
| `smokealarm-trader` | (same) | Possibly legacy |
| `smokealarm-superadmin` | (same) | Possibly legacy |
| `sensor-global-cloudtrail-logs` | CloudTrail audit log archive | **Prod** |
| `sensor-global-prod-alb-logs` | ALB access logs | **Prod** |
| `sensor-global-prod-cloudfront-logs` | CloudFront access logs | **Prod** |
| `sensoralblogs` | ALB log catchall | **Prod** |
| `smokebucketlogs` | S3 server access logs | **Prod** |
| `smokesnslogs` | SNS logs | **Prod/shared** |
| `smoke-system-manager-log` | Systems Manager logs | **Shared** |
| `config-bucket-747293622182` | AWS Config snapshots | **Shared** |
| `sensor-global-cloudtrail-logs` | CloudTrail | **Shared** |
| `sensor-newrelic-firehose-17c83850` | New Relic Firehose (can be removed) | **Shared** |
| `sensor-securityhub-findings` | Security Hub exports | **Shared** |
| `sensor-mysql-snapshot-temp` | Temporary RDS snapshot transfers | **Ops** |
| `codepipeline-ap-southeast-2-85960498242` | CodePipeline artifacts | **CI/CD** |

Non-prod buckets (QA, dev, sandbox, UAT) are out of scope for migration.

### S3 migration plan update

1. **`smokeapibucket`** (3.4 GB): Sync to new account bucket using `aws s3 sync` with `--source-region ap-southeast-2` or via a cross-account IAM role. This is the most important and largest bucket.
2. **`smokealarmprod-admin`** (24 MB): Simply redeploy the Angular app via CodePipeline to a new S3 bucket in the new account. The 24MB is a compiled build artifact, not user data.
3. Log/audit buckets: Set up new buckets in new account; do not migrate old logs (they stay in source account for the soak period).
4. `sensor-newrelic-firehose-*`: Do not migrate — New Relic is being dropped.

---

## INV-05: IAM Users, Roles, and Secrets Audit

### IAM users (current account)

| Category | Users | Migration notes |
|---|---|---|
| **Current SensorGlobal staff** | `tom.mcevoy@sensorglobal.com`, `peresada1@gmail.com`, `Pawan_Sensor` | Recreate in new account |
| **Appinventiv dev team** | `Appinventiv_S3`, `deepak.yadav@appinventiv.com`, `gurpreet.singh2@appinventiv.com`, `jeetendra.singh@appinventiv.com`, `lokendra.saini@appinventiv.com`, `rudresh.kumar@appinventiv.com` | Decision needed: does JV retain Appinventiv? |
| **CI/CD** | `bitrise` | Recreate for Bitrise CI/CD integration |
| **Service accounts** | `boto3`, `lambda-fun-sandbox`, `s3-dev_usr`, `S3-uat-usr`, `sandbox-user`, `secret-manager-local-sso`, `secrete-manager`, `cloudtrail-CMK-User`, `security-hub-finding`, `SNS_S3_USER_QA`, `SNS_USER_DEV`, `odoo-dev` | Recreate only those needed for prod |
| **Monitoring** | `zabbix_monitoring` | See note below |
| **External/MSP** | `MSP-user` | Decision needed |

**Note on Zabbix:** The `zabbix_monitoring` IAM user indicates Zabbix is (or was) used as a monitoring tool alongside New Relic. This was not previously identified. Confirm with the ops team whether Zabbix is still active.

### IAM instance profiles

Only **one production application instance profile** identified: `smoke-alaram-api-logs-role`. This single role is shared across all 5 prod API servers. The role name suggests it was originally created for CloudWatch log shipping.

The SSO, keychain, EMQX, Kafka, Odoo, and WordPress servers likely rely on the same role or have no instance profile (relying on hardcoded credentials from Secrets Manager).

### Secrets Manager inventory (relevant to migration)

**Application secrets (must be recreated in new account):**

| Secret | Contents |
|---|---|
| `sensor-prod` | ~100 key-value pairs: DB credentials, MQTT credentials, Redis endpoint, MongoDB Atlas URL, all third-party API keys (KORE, Twilio, SendGrid, PropertyMe, PropertyTree, Sentry, LaunchDarkly, Google reCAPTCHA, Odoo, NEXU, Corpsure) |
| `sensor-prod-sso` | SSO provider configuration |
| `sensor-qa`, `sensor-qa-sso` | QA — out of scope |
| `sensor-dev-sso`, `sensor-sandbox`, `sensor-sandbox-sso`, `sensor-uat`, `sensor-uat-sso`, `smoke-dev`, `local`, `local-dev-sso` | Non-prod — out of scope |

**Third-party credentials found in `sensor-prod` secret (full re-onboarding checklist):**

| Service | Credential found | Action at migration |
|---|---|---|
| AWS access keys | `AWS_ACCESS_KEY`, `AWS_SECRET_KEY` (IAM user creds) | Replace with IAM role or new account IAM user creds |
| AWS Lambda access keys | `AWS_LAMBDA_ACCESS_KEY`, `AWS_LAMBDA_SECRET_KEY` | Replace with new account IAM role |
| MySQL | `DB_MYSQL_USERNAME/PASSWORD` | Update endpoint + credentials |
| MongoDB Atlas | `MONGO_DB_URL`, `MONGO_DB_ARCHIVE_URL` | Update to new Atlas cluster URL |
| Redis | `REDIS_HOST` | Update to new ElastiCache endpoint |
| SendGrid | `EMAIL_PASSWORD`, `SENDGRID_API_KEY`, `SENDGRID_API_KEY_FOR_READING` | Re-register sender domain for new account, or reuse same key |
| Twilio | `TWILIO_ACCOUNT_SID`, `TWILIO_AUTH_TOKEN` | Update to new number if required |
| KORE Wireless | `KORE_SECRET`, `KORES_API_AUTH_URL`, `KORES_API_CLIENT_ID`, `KORES_API_SECRET` | Must re-register for new JV entity |
| PropertyMe | `PROPERTYME_API_CLIENT_ID`, `PROPERTYME_API_CLIENT_SECRET` | Re-register OAuth app callback URL |
| PropertyTree | `PROPERTYTREE_APPLICATION_KEY`, `PROPERTYTREE_SUBSCRIPTION_KEY` | Verify if re-registration needed |
| Console Cloud | `CONSOLE_CLOUD_API_CLIENT_ID`, `CONSOLE_CLOUD_API_SECRET` | Verify if re-registration needed |
| Sentry | `SENTRY_DNS_URL` | Create new Sentry project or reuse |
| LaunchDarkly | `LAUNCH_DARKLY_SDK_KEY` | Update if tied to old account |
| Google reCAPTCHA | `RECAPTCHA_URLSECRET_KEY`, `RECAPTCHA_API_KEY`, `RECAPTCHA_PROJECTID` | Update project ID for new GCP project |
| Odoo | `ODOO_BASICAUTH_USERNAME/PASSWORD` | Update after Odoo migration |
| NEXU | `NEXU_API_TOKEN` | Verify if re-registration needed |
| Corpsure | `CORPSURE_LOGIN_USERNAME/PASSWORD` | Verify if re-registration needed |
| New Relic | `NEW_RELIC_LICENSE_KEY` | **Remove** — New Relic being dropped |

**Security note:** The `sensor-prod` secret contains hardcoded IAM access keys (`AWS_ACCESS_KEY`, `AWS_SECRET_KEY`). These are long-term credentials that bypass IAM role-based access. These should be replaced with instance role credentials in the new account — not carried over as-is.

---

## INV-06: DNS Strategy and Domain Registrar

### Domain registrar

All 14 domains are registered via **AWS Route 53 Registrar** (confirmed via `aws route53domains list-domains`). This is the simplest possible scenario — domain transfer between AWS accounts is a Route 53 Registrar operation and does not require involvement of an external registrar.

### Transfer lock status

| Domain | TransferLock | Expiry |
|---|---|---|
| `sensorglobal.com` | **Locked** | 2030-09-09 (safe) |
| `sensoriq.co.uk` | **Locked** | 2026-11-30 |
| All other 12 domains | Unlocked | Various |

Transfer locks must be disabled before a domain can be transferred. For Route53-to-Route53 transfer this is done via the AWS console.

### Domains expiring within 12 months (risk)

These domains are on AutoRenew but should be confirmed before decommissioning the source account:

| Domain | Expiry |
|---|---|
| `sensorglobal.au` | **2026-06-12** — 2 months away |
| `sensorglobal.co.uk` | **2026-07-07** — 3 months away |
| `sensorglobal.com.au` | **2026-07-07** — 3 months away |
| `sensorinsure.com` | **2026-09-03** — 5 months away |
| `sensorinsure.com.au` | **2026-09-03** — 5 months away |
| `sensorglobal.net` | **2026-06-12** — 2 months away |

AutoRenew is set on all — verify that a valid payment method is attached to the source account before it is decommissioned. If the account is closed, AutoRenew may fail.

### Key DNS records in `sensorglobal.com` (119 total)

**Production endpoints (direct IPs — must update at cutover):**

| Record | Type | Current value | Notes |
|---|---|---|---|
| `mqtt.sensorglobal.com` | A | `13.211.102.112` | Emqx-prod public IP — direct IP record |
| `kafka.sensorglobal.com` | A | `13.237.253.114` | prod-kafka public IP — direct IP record |
| `sensorglobal.com` | A | `54.253.158.197` | Odoo-production public IP |
| `crm.sensorglobal.com` | A | `54.253.158.197` | Odoo-production (same IP) |
| `api.sensorglobal.com` | CNAME | ALB DNS | `smokeapi-lb-898969882...elb.amazonaws.com` |
| `auth.sensorglobal.com` | CNAME | ALB DNS | `sso-api-lb-1719844694...elb.amazonaws.com` |
| `keychain.sensorglobal.com` | CNAME | ALB DNS | `keychain-production-alb-496662693...elb.amazonaws.com` |
| `admin/agent/trader/owner/activation/reschedule.sensorglobal.com` | CNAME | CloudFront | `d2obn6q80a5yi1.cloudfront.net` |

**Note on `keychain.*`:** All environments (`keychain.`, `keychain-development.`, `keychain-qa.`, `keychain-sandbox.`) point to the **same ALB** (`keychain-production-alb-496662693`). This means ALL environments share one keychain service. The new account must provide a keychain service before any environment's DNS is updated.

**Email records:**
- MX → Microsoft 365 (`mail.protection.outlook.com`) — primary corporate email
- SendGrid DKIM for transactional email (`u23692234.wl235.sendgrid.net`) and compliance email
- `DMARC p=none` — not enforced; should be strengthened to `p=quarantine` in the new account

### Recommended DNS migration strategy

Given all domains are in Route 53:

1. **Do not transfer hosted zones before cutover.** Keep `sensorglobal.com` hosted zone in the source account during the entire migration.
2. At each cutover step, update the specific A/CNAME records within the source account's hosted zone to point at new account resources.
3. After final validation (Phase 7 soak period), transfer the hosted zones to the new account via Route 53 zone migration.
4. Transfer domain registrations from source account to new account using Route 53 Registrar transfer.
5. Disable TransferLock on `sensorglobal.com` and `sensoriq.co.uk` before initiating the transfer.

**Ensure AutoRenew payment method is valid** on the source account throughout the migration and soak period to prevent domain expiry.

---

## INV-07: CI/CD and Deployment Automation

### CI/CD build tool: **Bitrise**

The IAM user `bitrise` exists in the account. No `bitrise.yml` was found in the checked code directories, but Bitrise is the external CI/CD platform. Builds are triggered in Bitrise and artifacts are pushed to CodePipeline or directly to CodeDeploy.

### CodeDeploy applications (11 total)

| Application | Deployment Group | Environment |
|---|---|---|
| `Smoke-API` | `Smoke-API` | **Production API** |
| `sso-api` | `sso-api-dg` | **Production SSO** |
| `Odoo-Addons` | `Odoo-Addons` | Production Odoo |
| `Smoke-development-API` | `Smokealarmdevelopmentapi` | Dev |
| `Smoke-Sandbox-API` | `Smoke-Sandbox-Api-Dg` | Sandbox |
| `Smoke-qa-API` | `Smoke-qa-API` | QA |
| `sensor-uat-api-codedeploy` | `uat-api-deployment-group` | UAT |
| `sandbox-sso-api` | `sandbox-sso-api-dg` | Sandbox SSO |
| `qa-sso-api` | `qa-sso-api-dg` | QA SSO |
| `uat-sso-codedeploy` | `uat-sso-dg` | UAT SSO |
| `dev-sso-api-dg` | `dev-sso-api-dg` | Dev SSO |

**Production API deployment type:** `BLUE_GREEN` with `WITH_TRAFFIC_CONTROL`. This means:
- Deployments create a new set of EC2 instances
- Load balancer swaps traffic to the new set
- Old instances are terminated after a configurable wait

This requires the ALB and target groups to be correctly pre-configured in the new account before CodeDeploy can run.

### CodePipeline (16 pipelines)

Includes `Smoke-production-admin` — this deploys the compiled Angular SPA to S3, then invalidates the CloudFront cache. This pipeline must be recreated in the new account pointing at the new S3 bucket and CloudFront distribution ID.

### AppSpec files

**`sensor-alarm-backend/appspec.yml`:**
```yaml
version: 0.0
os: linux
files:
  - source: /
    destination: /home/ec2-user/smoke_api
hooks:
  AfterInstall:
    - location: scripts/install-dependencies.sh    # runs npm install
      timeout: 600
  ApplicationStart:
    - location: scripts/start-server.sh            # starts PM2
      timeout: 600
```

**`sso-provider/appspec.yml`:** Same structure, deploys to `/home/ec2-user/sso-api`.

Both are simple and portable. No account-specific values are hardcoded in the AppSpec itself. Account-specific values (S3 bucket ARN for the deployment artifact, CodeDeploy service role ARN) are in the CodeDeploy configuration, not in the AppSpec.

### What must be recreated in the new account

1. CodeDeploy application: `Smoke-API` with BLUE_GREEN deployment group and ALB integration
2. CodeDeploy application: `sso-api` with deployment group
3. CodeDeploy application: `Odoo-Addons` (if Odoo is being migrated)
4. CodePipeline: `Smoke-production-admin` (update S3 bucket name and CloudFront distribution ID)
5. IAM role for CodeDeploy with permissions to push to the new account's EC2 and S3
6. Bitrise: update the IAM user/OIDC configuration to target the new account

---

## INV-08: Active User and Device Headcount

### RDS DatabaseConnections (proxy for active users)

7-day trend (2026-04-03 to 2026-04-10):

| Date | Average connections | Peak |
|---|---|---|
| 2026-04-03 | 4.3 | 39 |
| 2026-04-04 | 4.3 | 40 |
| 2026-04-05 | 4.4 | 48 |
| 2026-04-06 | 4.4 | 64 |
| 2026-04-07 | 4.6 | 48 |
| 2026-04-08 | 4.7 | 44 |
| 2026-04-09 | 4.4 | 43 |
| 2026-04-10 | 4.3 | 39 |

**Interpretation:** Average of ~4.3 simultaneous DB connections at any moment with peaks of ~40-64. With 5 API servers running connection pools, this implies normal background polling load. The peak-to-average ratio of ~10:1 suggests occasional batch operations or alarm processing spikes. The low average confirms this is not a high-throughput OLTP system — it is an event-driven IoT platform with moderate query load.

### MQTT connected device count

Cannot be directly queried without EMQX SSH access. Proxy estimate from prior investigation: EMQX `NetworkIn` shows ~750KB–1MB/hr of sustained MQTT traffic 24/7. With MQTT keep-alive packets and small alarm payloads, this is consistent with hundreds to low-thousands of connected hubs.

A direct count requires: `emqx_ctl clients list | wc -l` (requires SSH via OpenVPN).

### API request rate (from prior investigation)

~8,300–8,600 HTTP requests/hour average. This implies:
- Automated background requests (hub polling, KORE SIM callbacks, PropertyMe sync) account for the majority
- Active user sessions are a small fraction of the total
- No severe scaling concerns for `t3.medium` API servers

### New account sizing recommendation

The current production sizing is appropriate for the observed load. Recommend maintaining the same instance types in the new account:
- 5× `t3.medium` API servers (or start with 3 and scale up if needed)
- 1× `t3.small` SSO server
- 1× `c5.xlarge` EMQX broker
- 1× `t3.medium` Kafka broker
- 1× `t3.small` Keychain service
- 1× `t2.micro` Cron server (see below)
- 1× `t3.large` Odoo
- 1× `t3.medium` WordPress

---

## Kafka Broker Deep Dive (SSH via SSM — 2026-04-11)

> Instance: `i-09d176fd791fe7c64` (`10.0.1.199`), accessed via `aws ssm start-session`.

### Version and mode

- **Version:** Kafka **3.0.0** (Scala 2.13) — from jar `kafka_2.13-3.0.0.jar`
- **Mode:** **ZooKeeper** (not KRaft) — `zookeeper.connect=localhost:2181`, ZooKeeper 3.6.3
- **Single broker** — `broker.id=0`, replication factor = 1, no HA

### Topics

Only **one application topic** exists:

| Topic | Partitions | Notes |
|---|---|---|
| `logs-production` | 1 | Only application topic. All sensor/alarm events. |
| `__consumer_offsets` | 50 | Internal Kafka bookkeeping — not migrated |

### Consumer group

- **Group name:** `smoke-alarm`
- **Members:** 5 active consumers (confirmed from GroupCoordinator logs, 2026-04-09)
- **Generation:** 62 at last rebalance — actively consuming
- These consumers are the 5 API servers subscribing to `logs-production`

### Broker configuration

| Setting | Value | Notes |
|---|---|---|
| `listeners` | `PLAINTEXT://ec2-13-237-253-114.ap-southeast-2.compute.amazonaws.com:9092` | Public hostname, no TLS, no auth |
| `log.dirs` | `/tmp/kafka-logs` | 🚨 **In `/tmp` — data lost on reboot** |
| `log.retention.hours` | `168` | 7-day retention |
| `log.segment.bytes` | `1073741824` | 1 GB segment size |
| `offsets.topic.replication.factor` | `1` | No redundancy |
| JVM heap | `-Xmx1G -Xms1G` | 1 GB heap |

### Disk usage

- `/tmp/kafka-logs/` total: **236 MB** (7 days of production events)

### 🚨 Critical issues

1. **`log.dirs=/tmp/kafka-logs`** — log data is stored in `/tmp`. A reboot or instance stop wipes all topic data and consumer offset checkpoints. The broker has been running since **2026-02-22** without a reboot (7 weeks), so this hasn't caused visible data loss yet.
2. **No TLS, no auth** — broker is `PLAINTEXT` only; anyone with network access to port 9092 can produce/consume.
3. **Kafka is not stateful for migration** — the `logs-production` topic is a streaming log, not authoritative state. Consumer offsets are managed by the group coordinator. Starting fresh on a new broker is safe — consumers reconnect and resume from `latest`.

### Migration implication

- **Do not attempt to migrate Kafka data.** Start fresh.
- Create `logs-production` topic with 1 partition, 7-day retention.
- Fix `log.dirs` to use a mounted EBS volume (e.g. `/data/kafka-logs`) on the new instance.
- Consider upgrading to KRaft mode (Kafka 3.x supports it, eliminates ZooKeeper dependency).
- No auth/TLS today — consider adding SASL_PLAINTEXT or mTLS in new account.

---

## Newly Discovered Services Not in Original Plan

The following services were discovered during this investigation and were not in the original Phase 3 rebuild plan:

| Service | EC2 | Notes |
|---|---|---|
| **`smoke-prod-api-cron-server`** | `t2.micro` (`i-0a1cd47cfadedf035`) | Dedicated cron job server. Must be added to Phase 3. |
| **`wordpress-prod`** | `t3.medium` (`i-0bd9a5e391e812fcc`) | WordPress for the public-facing website (not Odoo). No public IP — behind ALB. Must check what ALB and DNS record serves it. |

**Action:** Update Phase 3 to include cron server and WordPress in the rebuild list.

---

## Summary of Plan-Changing Findings

| Finding | Impact on migration plan |
|---|---|
| MongoDB is Atlas (not EC2) | Phase 2.2: switch from EC2 migration to Atlas cluster migration (Live Migration or mongodump) |
| Redis cold-start is safe | Phase 2.4: no warm migration needed; just provision fresh ElastiCache in new account |
| All prod portals in ONE S3 bucket (24 MB) | Phase 3.8: frontend migration is just a re-deploy, not a data migration |
| `smokeapibucket` is 3.4 GB | Phase 2.5: must sync this bucket via cross-account S3 sync |
| CodeDeploy is BLUE_GREEN | Phase 3.3: ALB + Target Groups must be pre-configured before first CodeDeploy run |
| Cron server exists | Phase 3: add `smoke-prod-api-cron-server` (t2.micro) to rebuild list |
| WordPress exists as separate service | Phase 3: add `wordpress-prod` (t3.medium) to rebuild list |
| KORE/ONO SIM IPs in EMQX SG (20 CIDRs) | Phase 3.1: copy all 20 CIDRs to new EMQX security group for both port 1883 and port 18756 |
| Developer direct-access SG rule (116.73.36.32/32) | Do NOT carry this to new account — use OpenVPN only |
| All environments share keychain ALB | New account keychain must be live before any DNS update touches keychain.* |
| Domains registered in Route53 Registrar | Use Route53 Registrar transfer (no external registrar involvement needed) |
| `sensorglobal.au`, `.net`, `.co.uk`, `.com.au` expire mid-2026 | Ensure AutoRenew payment is valid before source account is closed |
| DMARC `p=none` | Strengthen to `p=quarantine` in new account |
| Zabbix monitoring (`zabbix_monitoring` IAM user) | Confirm if Zabbix is still active; if so, include in monitoring migration |
| Bitrise is CI/CD (not GitHub Actions) | Update Bitrise IAM user/OIDC to target new account |
| IAM hardcoded access keys in secrets | Do not carry `AWS_ACCESS_KEY`/`AWS_SECRET_KEY` from secret to new account — replace with IAM roles |
| 14 domains need payment protection during soak period | Note in decommission plan: confirm payment method before closing source account |
| Kafka `log.dirs=/tmp/kafka-logs` | **Source broker bug**: data lost on reboot. New broker must use EBS-backed volume. No impact on migration (Kafka is not stateful). |
| Kafka no auth/TLS | Source broker is `PLAINTEXT` only. New account should add SASL or mTLS — optional blocker. |
| Kafka topic inventory: 1 topic (`logs-production`) | Phase 3.2: create exactly 1 topic with 1 partition, 7-day retention. No data migration needed. |
| **🚨 ACM wildcard cert for `*.sensorglobal.com` EXPIRED 2026-04-09** | All 4 CloudFront distributions (prod, UAT, QA, sandbox, dev web apps) and 7 ALBs (incl. `wordpress-prod-alb`, `sso-api-lb`) are serving an expired cert. Zero valid certs in `us-east-1`. **Immediate fix:** create Amazon-managed ACM certs (DNS-validated via Route 53) in both `us-east-1` and `ap-southeast-2`; update CloudFront and ALB listeners — zero downtime, no cost. These certs auto-renew. New account should start with Amazon-managed certs from day one. Full details: `investigations/2026-04-12-certificate-expiry-incident.md` |
| `sensor-prod` MySQL custom parameter group | Phase 2.1: recreate `prod-mysql-8-parameter-group` with `group_concat_max_len=100000` and `log_bin_trust_function_creators=1` before restoring — app likely breaks without these |
| `sensor-prod` storage is encrypted (KMS) | Phase 2.1: must share KMS key cross-account before snapshot can be restored in target account |
| `odoo-production` io1 storage oversized | Phase 2.3: switch to gp3 in new account — same 3000 IOPS, ~65% cost reduction (~$215/month saving) |
| `odoo-production` has no Multi-AZ | Phase 2.3: decision needed — Single-AZ acceptable or upgrade? Recommend Single-AZ unless Odoo SLA requires it |
