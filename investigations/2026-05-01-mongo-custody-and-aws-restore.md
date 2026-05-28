# MongoDB Custody And AWS Restore Plan

- Date: 2026-05-01
- Scope: risk mitigation for MongoDB Atlas ownership uncertainty and AWS-owned recovery options
- Related systems: MongoDB Atlas `sensor-prod`, Atlas Online Archive, AWS account `747293622182`, S3, KMS, Secrets Manager, EC2 SSM, Amazon DocumentDB

## Objective

Define a safe, AWS-side recovery plan in case the current MongoDB Atlas cluster becomes inaccessible because Atlas organization ownership is unclear after the JV transition.

The goal is not to replace Atlas immediately. The goal is to ensure the JV can recover production MongoDB data from assets controlled by AWS account `747293622182`, then choose the lowest-risk long-term target.

No AWS or Atlas mutation was performed during this investigation.

## Sources Used

- Repository files:
  - `AGENTS.md`
  - `docs/investigations/2026-04-10-inv-results.md`
  - `docs/investigations/2026-04-10-migration-plan.md`
  - `docs/investigations/2026-04-17-prod-snapshot-coverage.md`
  - `docs/runbooks/07-local-db-access.md`
  - `code/sensor-alarm-backend/src/utils/BootStrap.ts`
  - `code/infra/terraform/environments/prod/locals.tf`
- AWS commands or consoles:
  - Prior read-only CloudWatch evidence on 2026-05-01 showed live prod Mongoose writes and `readyState >>> 1`
  - No AWS mutations were run for this note
- External references:
  - MongoDB Database Tools `mongodump`: https://www.mongodb.com/docs/database-tools/mongodump/
  - MongoDB Database Tools `mongorestore`: https://www.mongodb.com/docs/database-tools/mongorestore/mongorestore-examples/
  - MongoDB Atlas AWS Marketplace billing: https://www.mongodb.com/docs/atlas/billing/aws-self-serve-marketplace/
  - MongoDB Atlas move project docs: https://www.mongodb.com/docs/atlas/tutorial/manage-projects/
  - Amazon DocumentDB dump/restore: https://docs.aws.amazon.com/documentdb/latest/developerguide/backup_restore-dump_restore_import_export_data.html
  - AWS CLI `docdb create-db-cluster`: https://docs.aws.amazon.com/cli/latest/reference/docdb/create-db-cluster.html

## Findings

### Current MongoDB State

- Production MongoDB is Atlas-managed, not EC2-hosted.
- Source cluster documented in the repo: `sensor-prod.vr48f.mongodb.net`.
- Source live database documented in the repo: `sensorproddb`.
- `MONGO_DB_URL` is stored in AWS Secrets Manager secret `sensor-prod`.
- `MONGO_DB_ARCHIVE_URL` points to Atlas Online Archive.
- `KEEP_DATA_ALIVE_IN_CURRENT_MONGODB` is documented as `90`, so the live cluster should contain the active data window and older data is handled by Atlas Online Archive.
- The backend opens both the live MongoDB connection and archive connection during startup. A target migration must account for both values.

### Ownership Reality

An AWS account can own S3, KMS, EC2, Secrets Manager, and Amazon DocumentDB resources. It does not truly own a MongoDB Atlas cluster unless the Atlas organization/project ownership is also controlled by JV users.

AWS Marketplace for MongoDB Atlas solves billing/procurement, not custody. It can make AWS pay the Atlas invoice, but it does not by itself transfer Atlas organization ownership.

### Safest Recommendation

Use a two-track risk mitigation:

1. **Immediate custody backup in AWS:** take an encrypted `mongodump` archive from the Atlas live database and store it in JV-controlled S3 with KMS encryption and checksums.
2. **Preferred recovery target:** restore-test into a new JV-controlled Atlas cluster first, because this preserves MongoDB compatibility and is the lowest application risk.
3. **AWS-owned fallback target:** create an Amazon DocumentDB pilot and restore the same dump only after compatibility and performance testing. Treat DocumentDB as a candidate fallback, not a drop-in replacement.

### Why Not Cut Directly To DocumentDB

DocumentDB is AWS-owned and may be attractive for custody, but it is not genuine MongoDB. MongoDB tooling and drivers may work for many operations, but compatibility must be proven against this application.

SensorSyn-specific areas to test before any DocumentDB cutover:

- Mongoose startup and connection settings.
- Log and audit aggregations.
- High-volume inserts into log collections.
- `countDocuments`, `updateMany`, `findOneAndUpdate`, and index-heavy queries.
- TTL/session behavior in the SSO provider.
- Failover behavior and read preference handling.
- I/O cost under realistic load.
- Archive behavior, because DocumentDB does not replace Atlas Online Archive.

## Risks Or Constraints

- Running `mongodump` against production is read-heavy and may affect the Atlas M10 cluster if run at peak traffic.
- A dump taken while writes are active is not a clean business cut unless oplog/live migration is explicitly designed. The recommended first production-grade dump uses a maintenance window.
- The dump contains production data and must be treated as sensitive.
- MongoDB connection strings include credentials. Avoid passing them directly in shell history or command lines where possible.
- Atlas Online Archive configuration and archived data are not captured by the live database dump.
- Any S3 bucket, KMS key, EC2 utility host, or DocumentDB cluster creation is an AWS mutation and must be performed by a human/operator under explicit approval. Codex must not run those changes under this repo's guardrails.

## Cost Optimization Opportunities

- Keep the AWS custody dump lifecycle-limited unless a long retention requirement is approved.
- Use an ephemeral utility EC2 instance and terminate it after the dump is verified.
- Use DocumentDB initially as a short-lived pilot to avoid committing to steady production cost before compatibility is proven.
- Prefer Atlas Marketplace or JV-owned Atlas if the business concern is billing/ownership and not an AWS-native database requirement.

## Recommended Next Steps

1. Use `docs/runbooks/11-mongo-custody-and-aws-restore.md` as the operator runbook.
2. Add two JV-controlled Atlas organization owners immediately if current Atlas access allows it.
3. Create the AWS custody bucket/KMS key and utility host under a formal change window.
4. Take and checksum a maintenance-window dump of `sensorproddb`.
5. Restore-test to a new JV-owned Atlas cluster.
6. Build a separate DocumentDB pilot only after the custody dump is secured.
7. Do not decommission or alter the old Atlas cluster until a target has passed restore and soak validation.

## Open Questions

- Can the JV obtain `Organization Owner` access in the current Atlas organization?
- Is the long-term requirement "JV owns Atlas" or "AWS account owns the database service"?
- How much archived data must be migrated, and is Atlas Online Archive acceptable after the ownership transition?
- What maintenance window is acceptable for a final cutover dump?
- Should the AWS-owned fallback be two-instance DocumentDB for standby/cost balance or three-instance DocumentDB for production-grade HA?
