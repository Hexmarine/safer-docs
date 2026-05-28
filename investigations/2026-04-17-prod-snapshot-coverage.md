# Prod Snapshot Coverage Investigation

**Date:** 2026-04-17  
**Context:** Pre-optim safety review — investigating backup coverage before applying `optim.tfvars` to prod.

---

## Summary

MongoDB is **fully on Atlas** (`sensor-prod.vr48f.mongodb.net`) — not on EC2. Atlas manages its own backups independently. All prod **RDS** instances have continuous automated backups (7-day retention). A `pre-optim-snapshot.sh` script was created to take explicit named snapshots before any prod Terraform apply.

---

## MongoDB is on Atlas (not EC2)

From the prod Secrets Manager secret (`sensor-prod`):
```
MONGO_DB_URL:         mongodb+srv://...@sensor-prod.vr48f.mongodb.net/...
MONGO_DB_ARCHIVE_URL: mongodb://...@atlas-online-archive-...  (Atlas Online Archive)
```

The `prod-kafka` EC2 (despite the misleading legacy name tag `prod-mongodb-kafka-server`) runs **Kafka only**. MongoDB was migrated to Atlas at some earlier point. Atlas manages its own backup/snapshot policies.

**Action:** Verify Atlas backup policy is enabled in the Atlas console. The EBS snapshot of `prod-kafka` is for Kafka data, not MongoDB.

---

## Data Stores Inventoried

| Store | Resource | Backup status |
|---|---|---|
| MongoDB | Atlas `sensor-prod.vr48f.mongodb.net` | ✅ Atlas-managed backups (verify policy in Atlas console) |
| MongoDB Archive | Atlas Online Archive | ✅ Atlas-managed |
| RDS MySQL `sensor-prod` | `db-JYLS6IJJMN...` | ✅ 7-day automated backup, latest restore: 2026-04-17T04:09Z |
| RDS Postgres `odoo-production` | `db-FP5J3ALYFF...` | ✅ 7-day automated backup, latest restore: 2026-04-17T04:04Z |
| ElastiCache Redis `smokeprodapiredis` | `cache.t3.medium ×2` | ⚠️ `snapshot_retention_limit=1` — only 1 daily snapshot retained |
| Kafka on `prod-kafka` EC2 | `vol-0868b82bb3f8329e4` (50GB) | ❌ DLM name mismatch — see below |
| `Emqx-prod` EC2 | `vol-01150988db7184afd` | ❌ Not covered by DLM |
| `wordpress-prod` EC2 | `vol-0688c0a932cebc352` | ❌ Not covered by DLM |
| `wordpress-sensor-insure` EC2 | `vol-0f2c4069881201962` | ❌ Not covered by DLM |
| `openvpn-prod` EC2 | `vol-0d7c3cd405ef83a4f` | ❌ Not covered by DLM — lower risk (config only) |
| `Power-BI-Gateway` EC2 | `vol-0cfdcc56eebc91ac6` | ❌ Not covered by DLM — lower risk |

---

## DLM Policy Name Mismatch (Kafka volume, not MongoDB)

**DLM policy `policy-0441001e0e7116959`** targets Name=`prod-mongodb-kafka-server` — but the EC2 is named `prod-kafka`. The EBS snapshot would cover Kafka data. Last DLM snapshot of this volume: **Nov 2024**.

### Fix options (requires approval)

**Option A:** Add tag `prod-mongodb-kafka-server` to `prod-kafka` EC2 (minimal change, no policy edit).  
**Option B:** Update DLM policy target tag to `prod-kafka`.

---

## DLM Actual Coverage (verified)

| Volume | Instance | Size |
|---|---|---|
| `vol-011a40eebb7e65b65` | `Smoke-prod-Keychain-app` | 20 GB |
| `vol-02159894f6a980b61` | `smoke-prod-api-cron-server` | 8 GB |
| `vol-014d9b17b0ed5277a` | `Odoo-production` | 150 GB |
| `vol-0dd46b1479a7c026b` | `prod-sso-api` | 30 GB |
| `vol-0d0627791dfcd6d78` | `SANDBOX-Kafka-mongo-Smoke` (not prod — oversight in policy) | 50 GB |

---

## Pre-Optim Snapshot Script

`scripts/pre-optim-snapshot.sh` — creates named snapshots with `pre-optim-YYYYMMDD-HHmmss` prefix for:
- RDS MySQL + Postgres (manual DB snapshots)
- ElastiCache Redis (manual cache snapshot)
- Kafka/prod-kafka EBS, Emqx-prod EBS, wordpress-prod EBS, wordpress-sensorinsure EBS

MongoDB is excluded — Atlas handles its own backups.

Run **before** `terraform apply -var-file=optim.tfvars` in prod.

---

## Note: optim.tfvars risk to data is minimal

The prod `optim.tfvars` changes (gp2→gp3, RDS gp2→gp3, log retention) are all **in-place performance/metadata changes**. AWS does not move or copy data during a gp2→gp3 conversion. The snapshots are a belt-and-suspenders precaution, not a response to data migration risk.
