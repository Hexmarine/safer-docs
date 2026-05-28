# Runbook 11 - MongoDB Custody Backup And AWS Restore

**Created:** 2026-05-01  
**Status:** Phase 0 verified by Codex; Phase 1 custody KMS/S3 and Phase 2 utility host applied externally by user and verified  
**Approval reference:** User requested implementation of the AWS-side Mongo custody and restore plan on 2026-05-01  
**Applies to:** MongoDB Atlas `sensor-prod`, AWS account `747293622182`, region `ap-southeast-2`  
**Primary evidence:** `docs/investigations/2026-05-01-mongo-custody-and-aws-restore.md`

## Purpose

Create a recoverable, AWS-controlled copy of the production MongoDB live database, then restore-test it into a target controlled by the JV.

This runbook separates three outcomes:

1. **Custody backup:** encrypted dump stored in the JV AWS account.
2. **Preferred recovery target:** new JV-owned MongoDB Atlas cluster.
3. **AWS-owned fallback:** Amazon DocumentDB pilot, gated by compatibility testing.

## Safety Rules

- Treat all steps as production-impacting.
- Do not run a production dump during peak traffic.
- Do not print MongoDB credentials in terminal logs, tickets, chat, screenshots, or docs.
- Do not delete, pause, resize, or modify the existing Atlas cluster during this runbook.
- Do not update `MONGO_DB_URL` or `MONGO_DB_ARCHIVE_URL` until the final cutover gate.
- Codex must not run the AWS mutation steps in this runbook under the repo guardrails. A human/operator must execute them.

## Recommended Defaults

| Item | Default |
|---|---|
| Region | `ap-southeast-2` |
| AWS account | `747293622182` |
| Source database | `sensorproddb` |
| Source secret | AWS Secrets Manager secret `sensor-prod` |
| Custody bucket | `sensor-prod-mongo-custody-747293622182` |
| KMS alias | `alias/sensor-prod-mongo-custody` |
| Dump runner | Temporary SSM-only EC2 utility host |
| Utility host subnets | private subnets `subnet-06f5ef2f572afe562`, `subnet-03de5b8459fa15e1b`, or `subnet-0b877428d609f42a3` |
| Utility host storage | 300 GB encrypted gp3 |
| Preferred restore target | New JV-owned Atlas M10, MongoDB 8.0, AWS Sydney |
| AWS-owned fallback target | Amazon DocumentDB 8.0, encrypted, deletion protection, two instances for pilot |

If the default S3 bucket name is unavailable globally, append the date suffix, for example `sensor-prod-mongo-custody-747293622182-20260501`.

## Execution Status

| Phase | Status | Verification |
|---|---|---|
| Phase 0 - Access and baseline | Complete | AWS caller, secret metadata, MongoDB tools, and prod Mongo log signals checked on 2026-05-01 |
| Phase 1 - AWS custody storage | Complete | User created KMS key `191ab3a4-a352-4d12-b057-610adae73a0c`; alias `alias/sensor-prod-mongo-custody` resolves; bucket `sensor-prod-mongo-custody-747293622182` has public access blocked, versioning enabled, and default KMS encryption |
| Phase 2 - Utility host | Complete | User launched `i-0c179962eb08ed4c6`; SSM reports `PingStatus=Online`, platform `Amazon Linux`, SSM agent `3.3.4108.0`; MongoDB tools installed after correcting repo URL; S3/KMS write test succeeded with SSE-KMS key `191ab3a4-a352-4d12-b057-610adae73a0c` |
| Phase 4 - AWS custody dump copy | Complete | User confirmed the completed production dump archive and checksum were uploaded to `s3://sensor-prod-mongo-custody-747293622182/dumps/final/`; retain `aws s3 ls` and `head-object` output when available for object size, version ID, and SSE-KMS verification |
| Phase 5 - New JV Atlas target | Complete | New Atlas cluster `sensor-prod` created in organization `Sensor Global Pty Ltd`, project `Sensor Production`; runner ping to new Atlas returned `ok: 1`; first restore attempt hit Atlas write-blocking on `tbl_sim_event_logs` due target disk exhaustion (`DiskUseThresholdExceeded`) after `7330707` documents restored and `293` failed; after increasing target storage, restore completed with `73138048` documents restored and `0` failed; target validation showed `16` collections, `1` view, `73137634` objects, `100` indexes, and write test passed |
| Phase 6 - Application cutover | Complete, archive follow-up pending | User updated `sensor-prod.MONGO_DB_URL` and `sensor-prod-sso.MONGODB_URI`, then restarted root-owned PM2 processes for API and SSO. Initial cutover URI values caused `MongoParseError: Password contains unescaped characters`; user corrected the URI/password values and restarted again with command `e33de6af-b10a-4b19-b152-95a3a82d9856`. Fresh incognito login then showed `client authentication failed`; Codex diagnosed both URIs were missing `/sensorproddb`. User corrected the DB path and restarted again with command `c824f9bc-ba25-461c-82f0-3c39fee15196`. Login then returned `400 VALIDATION_ERROR`; Codex compared old vs new Atlas `basemodels` indexes and found an extra unique `payload.grantId_1` index on the new cluster. User dropped that extra index, after which browser login succeeded. Redacted checks show both URIs now target `sensor-prod.yujwyg.mongodb.net/sensorproddb`; logs show `Connected to MONGO_DB in bootstrap`, no fresh `MongoParseError`, and no fresh SSO `client authentication failed` entries in the checked window. `MONGO_DB_ARCHIVE_URL` remains pointed at the old Atlas Online Archive endpoint and post-restart logs showed `MONGO_DB_ARCHIVED` connection errors, so archive recreation/retargeting remains pending |

## Phase 0 - Access And Baseline

Current read-only status from 2026-05-01:

| Check | Result |
|---|---|
| AWS identity | Account `747293622182`, caller `arn:aws:iam::747293622182:user/peresada1@gmail.com` |
| `sensor-prod` secret metadata | Readable; last changed `2025-04-30T20:30:11.535000+10:00`; AWS-managed/default key (`KmsKeyId: null`) |
| `mongodump` | Installed, version `100.16.0` |
| `mongorestore` | Installed, version `100.16.0` |
| `mongosh` | Installed, version `2.6.0` |
| Prod Mongo connectivity signal | `smoke-api-prod-pm2-out-log` contained recent `mongoose.connection.readyState >>> 1` entries |
| Prod Mongo error signal | No recent `MongooseServerSelectionError` found in `smoke-api-prod-pm2-error-log` for the checked window |

Read-only checks:

```bash
aws sts get-caller-identity
aws secretsmanager describe-secret --region ap-southeast-2 --secret-id sensor-prod
mongodump --version
mongorestore --version
mongosh --version
```

Confirm Atlas connectivity from production evidence before making any changes:

```bash
aws logs filter-log-events \
  --region ap-southeast-2 \
  --log-group-name smoke-api-prod-pm2-out-log \
  --start-time "$(date -u -d '24 hours ago' +%s000)" \
  --filter-pattern '"mongoose.connection.readyState >>> 1"' \
  --max-items 10
```

Decision gates:

- At least two JV-controlled people can access AWS account `747293622182`.
- The dump window is approved.
- The current Atlas cluster remains untouched.
- The restore target is created before the final production dump.

## Phase 1 - Create AWS Custody Storage

All commands in this phase are AWS mutations and require human execution.

Create a KMS key:

```bash
aws kms create-key \
  --region ap-southeast-2 \
  --description "SensorSyn production MongoDB custody dump encryption key" \
  --tags TagKey=System,TagValue=MongoDB TagKey=Purpose,TagValue=CustodyBackup

aws kms create-alias \
  --region ap-southeast-2 \
  --alias-name alias/sensor-prod-mongo-custody \
  --target-key-id <kms-key-id-from-create-key>
```

Create the S3 bucket:

```bash
aws s3api create-bucket \
  --region ap-southeast-2 \
  --bucket sensor-prod-mongo-custody-747293622182 \
  --create-bucket-configuration LocationConstraint=ap-southeast-2

aws s3api put-public-access-block \
  --region ap-southeast-2 \
  --bucket sensor-prod-mongo-custody-747293622182 \
  --public-access-block-configuration BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true

aws s3api put-bucket-versioning \
  --region ap-southeast-2 \
  --bucket sensor-prod-mongo-custody-747293622182 \
  --versioning-configuration Status=Enabled

aws s3api put-bucket-encryption \
  --region ap-southeast-2 \
  --bucket sensor-prod-mongo-custody-747293622182 \
  --server-side-encryption-configuration '{
    "Rules": [{
      "ApplyServerSideEncryptionByDefault": {
        "SSEAlgorithm": "aws:kms",
        "KMSMasterKeyID": "alias/sensor-prod-mongo-custody"
      },
      "BucketKeyEnabled": true
    }]
  }'
```

Add a lifecycle rule after the retention requirement is approved. Suggested default:

- `dumps/test/`: expire after 14 days.
- `dumps/final/`: transition to Glacier Instant Retrieval after 30 days, retain 180 days unless legal/compliance needs longer.

## Phase 2 - Create AWS Utility Host

All commands in this phase are AWS mutations and require human execution.

Recommended properties:

- SSM Session Manager only.
- No inbound SSH.
- Private subnet with NAT egress.
- IAM instance profile allows:
  - `secretsmanager:GetSecretValue` on `sensor-prod`
  - `s3:PutObject`, `s3:GetObject`, `s3:ListBucket` on the custody bucket
  - `kms:Encrypt`, `kms:Decrypt`, `kms:GenerateDataKey` on the custody KMS key
- 300 GB encrypted gp3 root or attached volume.
- MongoDB Database Tools installed.

Use the existing private subnets from prod Terraform locals:

- `subnet-06f5ef2f572afe562`
- `subnet-03de5b8459fa15e1b`
- `subnet-0b877428d609f42a3`

Do not use production API instances as the dump runner unless the team explicitly accepts disk, CPU, and operational coupling risk.

## Phase 3 - Prepare Secrets Safely On The Utility Host

Run on the utility host. Avoid putting the Mongo URI directly in shell history.

```bash
set +o history
umask 077
sudo install -d -m 700 /run/sensorglobal-mongo

aws secretsmanager get-secret-value \
  --region ap-southeast-2 \
  --secret-id sensor-prod \
  --query SecretString \
  --output text \
  | python3 -c 'import json,sys; print("uri: " + json.dumps(json.load(sys.stdin)["MONGO_DB_URL"]))' \
  | sudo tee /run/sensorglobal-mongo/mongodump.yml >/dev/null

sudo chmod 600 /run/sensorglobal-mongo/mongodump.yml
set -o history
```

Connectivity check:

```bash
mongosh --config /run/sensorglobal-mongo/mongodump.yml \
  --eval 'db.adminCommand({ ping: 1 })'
```

## Phase 4 - Take The Maintenance-Window Dump

Before starting:

- Put API, SSO, and cron writes into maintenance mode or stop write-producing services.
- Confirm no deployments or batch jobs are running.
- Record the UTC start time.

Run:

```bash
export DUMP_TS="$(date -u +%Y%m%dT%H%M%SZ)"
export DUMP_FILE="/mnt/mongo-dump/sensorproddb-${DUMP_TS}.archive.gz"
sudo mkdir -p /mnt/mongo-dump

mongodump \
  --config /run/sensorglobal-mongo/mongodump.yml \
  --db sensorproddb \
  --archive="${DUMP_FILE}" \
  --gzip \
  --numParallelCollections=4

sha256sum "${DUMP_FILE}" | tee "${DUMP_FILE}.sha256"
ls -lh "${DUMP_FILE}" "${DUMP_FILE}.sha256"
```

Upload to AWS custody storage:

```bash
aws s3 cp "${DUMP_FILE}" \
  "s3://sensor-prod-mongo-custody-747293622182/dumps/final/$(basename "${DUMP_FILE}")" \
  --sse aws:kms \
  --sse-kms-key-id alias/sensor-prod-mongo-custody

aws s3 cp "${DUMP_FILE}.sha256" \
  "s3://sensor-prod-mongo-custody-747293622182/dumps/final/$(basename "${DUMP_FILE}.sha256")" \
  --sse aws:kms \
  --sse-kms-key-id alias/sensor-prod-mongo-custody
```

Verify custody copy:

```bash
aws s3 ls "s3://sensor-prod-mongo-custody-747293622182/dumps/final/"
aws s3api head-object \
  --bucket sensor-prod-mongo-custody-747293622182 \
  --key "dumps/final/$(basename "${DUMP_FILE}")"
```

## Phase 5 - Restore To New JV-Owned Atlas

Create the target Atlas cluster in the JV-controlled Atlas organization:

- Region: AWS Sydney `ap-southeast-2`
- MongoDB version: 8.0
- Tier: at least M10 for first restore rehearsal
- Database name: keep `sensorproddb` unless there is a deliberate app config reason to change it
- Add Network Access for the utility host public egress IP, or use private connectivity if configured

Prepare the new Atlas URI config on the utility host:

```bash
set +o history
umask 077
sudo tee /run/sensorglobal-mongo/mongorestore-atlas.yml >/dev/null <<'EOF'
uri: "<NEW_MONGO_DB_URL>"
EOF
sudo chmod 600 /run/sensorglobal-mongo/mongorestore-atlas.yml
set -o history
```

Restore:

```bash
mongorestore \
  --config /run/sensorglobal-mongo/mongorestore-atlas.yml \
  --archive="${DUMP_FILE}" \
  --gzip \
  --nsInclude="sensorproddb.*"
```

If restoring into a different database name:

```bash
mongorestore \
  --config /run/sensorglobal-mongo/mongorestore-atlas.yml \
  --archive="${DUMP_FILE}" \
  --gzip \
  --nsFrom="sensorproddb.*" \
  --nsTo="<new-db-name>.*"
```

Validate counts and indexes:

```bash
mongosh --config /run/sensorglobal-mongo/mongodump.yml --quiet --eval '
const dbn = db.getSiblingDB("sensorproddb");
dbn.getCollectionNames().sort().forEach(c => print(c + "," + dbn.getCollection(c).estimatedDocumentCount()));
' > /tmp/source-counts.csv

mongosh --config /run/sensorglobal-mongo/mongorestore-atlas.yml --quiet --eval '
const dbn = db.getSiblingDB("sensorproddb");
dbn.getCollectionNames().sort().forEach(c => print(c + "," + dbn.getCollection(c).estimatedDocumentCount()));
' > /tmp/target-counts.csv

diff -u /tmp/source-counts.csv /tmp/target-counts.csv
```

Create a new Atlas Online Archive policy in the JV Atlas project before production cutover. The live dump does not capture the archive configuration or older archived records.

## Phase 6 - Optional Amazon DocumentDB Pilot

This is an AWS-owned fallback path, not the recommended first production cutover.

All commands in this phase are AWS mutations and require human execution.

Create a subnet group:

```bash
aws docdb create-db-subnet-group \
  --region ap-southeast-2 \
  --db-subnet-group-name sensor-prod-docdb-subnets \
  --db-subnet-group-description "SensorSyn production DocumentDB pilot subnet group" \
  --subnet-ids subnet-0ec989f2f61cb5f25 subnet-0949bfb4548081f25 subnet-0594cfdc32c9dbe6c
```

Create a security group allowing only app/SSO/cron/utility access on port `27017`:

```bash
aws ec2 create-security-group \
  --region ap-southeast-2 \
  --group-name sensor-prod-docdb-pilot \
  --description "SensorSyn production DocumentDB pilot access" \
  --vpc-id vpc-03ec09702866ca7b1

export DOCDB_SG_ID="<security-group-id-from-create-security-group>"

aws ec2 authorize-security-group-ingress \
  --region ap-southeast-2 \
  --group-id "${DOCDB_SG_ID}" \
  --ip-permissions '[
    {
      "IpProtocol": "tcp",
      "FromPort": 27017,
      "ToPort": 27017,
      "UserIdGroupPairs": [
        {"GroupId": "sg-0830a4cf088cc54be", "Description": "prod API servers"},
        {"GroupId": "sg-0d544dd0240803824", "Description": "prod SSO API"},
        {"GroupId": "sg-0bcf74ddca13e8370", "Description": "prod cron"},
        {"GroupId": "<utility-host-security-group-id>", "Description": "mongo dump utility host"}
      ]
    }
  ]'
```

Create the cluster:

```bash
aws docdb create-db-cluster \
  --region ap-southeast-2 \
  --db-cluster-identifier sensor-prod-docdb-pilot \
  --engine docdb \
  --engine-version 8.0.0 \
  --master-username sensoradmin \
  --manage-master-user-password \
  --db-subnet-group-name sensor-prod-docdb-subnets \
  --vpc-security-group-ids "${DOCDB_SG_ID}" \
  --storage-encrypted \
  --kms-key-id alias/sensor-prod-mongo-custody \
  --backup-retention-period 7 \
  --storage-type standard \
  --enable-cloudwatch-logs-exports audit profiler \
  --deletion-protection
```

Create two pilot instances:

```bash
aws docdb create-db-instance \
  --region ap-southeast-2 \
  --db-cluster-identifier sensor-prod-docdb-pilot \
  --db-instance-identifier sensor-prod-docdb-pilot-1 \
  --db-instance-class db.t4g.medium \
  --engine docdb

aws docdb create-db-instance \
  --region ap-southeast-2 \
  --db-cluster-identifier sensor-prod-docdb-pilot \
  --db-instance-identifier sensor-prod-docdb-pilot-2 \
  --db-instance-class db.t4g.medium \
  --engine docdb
```

Restore the same dump only after connectivity and TLS settings are confirmed for DocumentDB. Do not update production secrets until the compatibility gates below pass.

DocumentDB compatibility gates:

- Backend starts successfully.
- SSO provider starts successfully.
- Core read flows pass.
- SIM event/log inserts pass.
- Audit/log aggregation screens pass.
- TTL/session cleanup behaves acceptably.
- `countDocuments`, `updateMany`, `findOneAndUpdate`, and index-heavy queries pass.
- Failover test is acceptable.
- I/O and latency are acceptable for a full traffic replay or production-like test.

## Phase 7 - Final Cutover

Only run this phase after a restore target has passed validation.

1. Freeze writes.
2. Take a final fresh dump and checksum.
3. Restore final dump into the selected target.
4. Validate counts and smoke tests.
5. Update AWS Secrets Manager:
   - `MONGO_DB_URL`
   - `MONGO_DB_ARCHIVE_URL` if the archive endpoint changes
   - `KEEP_DATA_ALIVE_IN_CURRENT_MONGODB=90`
6. Restart API, SSO, and cron services.
7. Watch CloudWatch logs for:
   - `MongooseServerSelectionError`
   - `MongoNetworkError`
   - `Error connecting to MongoDB`
   - `mongoose.connection.readyState >>> 1`
8. Keep old Atlas cluster untouched for the soak window.

## Rollback

Rollback trigger:

- App cannot start.
- Login/SSO fails.
- Core reads fail.
- New Mongo connection errors appear.
- Counts do not match.
- Latency or error rate breaches the agreed threshold.

Rollback steps:

1. Freeze writes immediately.
2. Restore the previous `MONGO_DB_URL` and `MONGO_DB_ARCHIVE_URL` values in Secrets Manager.
3. Restart API, SSO, and cron.
4. Confirm prod logs show `mongoose.connection.readyState >>> 1`.
5. Preserve failed target logs and restore logs for reconciliation.
6. Do not delete the failed target until the write window has been reconciled.

## Pass Criteria

- AWS custody dump exists in S3 with checksum.
- Restore target contains expected collections and matching counts.
- Index restore errors are absent or understood and remediated.
- Backend, SSO, and cron start.
- Core production flows pass.
- No Mongo connection errors in the first 30 minutes.
- Old Atlas cluster remains available for rollback through the soak period.

## Notes

- AWS Marketplace for Atlas can move billing to AWS, but it does not by itself transfer Atlas organization ownership.
- A JV-owned Atlas organization is the safest compatibility target.
- Amazon DocumentDB is the AWS-owned database option, but it requires a compatibility and performance pilot before production cutover.
