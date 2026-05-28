# Runbook 02 — RDS odoo-production: io1 → gp3

**Proposal 2** from `docs/investigations/2026-04-11-cost-optimization-proposals.md`

- **Expected saving:** ~$330/month (elimination of Provisioned IOPS charges)
- **Risk:** Low — gp3 delivers the same 3,000 IOPS at no extra cost; migration is online
- **Compliance:** ✅ Storage type change does not affect data integrity or retention
- **Approval required:** Written approval + agreed maintenance window

---

## Context

`odoo-production` is an RDS instance using io1 (Provisioned IOPS) storage:
- 100 GB allocated, only ~10 GB in use
- 3,000 IOPS provisioned at $0.11/GB-month + $0.10/IOPS-month = **~$330/month in PIOPS alone**
- gp3 baseline is 3,000 IOPS included in the base price (~$0.115/GB-month, no IOPS surcharge)
- Migration is performed by RDS with no downtime on Multi-AZ instances

**Prerequisite: confirm actual IOPS usage is ≤3,000** (so gp3 baseline is sufficient)

```bash
# Check if Performance Insights is enabled
aws --profile sensorsyn-mfa rds describe-db-instances \
  --db-instance-identifier odoo-production \
  --query "DBInstances[0].{PI:PerformanceInsightsEnabled,Storage:StorageType,IOPS:Iops,Class:DBInstanceClass}" \
  --output table --region ap-southeast-2
```

If Performance Insights is enabled, check actual IOPS:
```bash
aws --profile sensorsyn-mfa pi get-resource-metrics \
  --service-type RDS \
  --identifier db:odoo-production \
  --start-time "$(date -u -d '-30 days' +%Y-%m-%dT%H:%M:%SZ)" \
  --end-time "$(date -u +%Y-%m-%dT%H:%M:%SZ)" \
  --period-in-seconds 86400 \
  --metric-queries '[{"Metric":"os.diskIO.rdsiops.read","GroupBy":{"Group":"db.host","Limit":1}},{"Metric":"os.diskIO.rdsiops.write","GroupBy":{"Group":"db.host","Limit":1}}]' \
  --region ap-southeast-2
```

**Decision rule:** If peak IOPS < 2,500 over the last 30 days → gp3 at 3,000 IOPS is safe.
If peak IOPS > 2,500 → gp3 can be provisioned with 3,000 IOPS explicitly (still cheaper than io1).

---

## Pre-change

```bash
bash scripts/health-check.sh --save-baseline tmp/baseline-rds-io1-gp3-$(date +%Y%m%d-%H%M%S).json
```

Also capture the current storage configuration:
```bash
aws --profile sensorsyn-mfa rds describe-db-instances \
  --db-instance-identifier odoo-production \
  --query "DBInstances[0].{StorageType:StorageType,IOPS:Iops,AllocatedStorage:AllocatedStorage,Status:DBInstanceStatus,MultiAZ:MultiAZ}" \
  --output table --region ap-southeast-2
```

---

## Approval required

- [ ] Written approval for this specific change from account owner
- [ ] Agreed maintenance window (recommended: Saturday 2am–4am AEST to minimise Odoo user impact)
- [ ] Odoo team notified of potential brief performance variation during migration

---

## Change command

**A human must run this after approval. Do not run without prior written approval.**

```bash
aws --profile sensorsyn-mfa rds modify-db-instance \
  --db-instance-identifier odoo-production \
  --storage-type gp3 \
  --iops 3000 \
  --apply-immediately \
  --region ap-southeast-2
```

**`--apply-immediately` note:** For Multi-AZ, the storage modification is performed on the
standby first, then the primary. There may be a brief failover (30–60 seconds). Schedule in
a maintenance window if Odoo users are active.

Monitor the modification progress:
```bash
watch -n 30 'aws --profile sensorsyn-mfa rds describe-db-instances \
  --db-instance-identifier odoo-production \
  --query "DBInstances[0].{Status:DBInstanceStatus,StorageType:StorageType,IOPS:Iops}" \
  --output table --region ap-southeast-2'
```

Expected status sequence: `available` → `modifying` → `available`
Typical duration: 15–45 minutes.

---

## Verification

```bash
bash scripts/health-check.sh --compare-to tmp/baseline-rds-io1-gp3-*.json
```

Confirm new storage type is gp3:
```bash
aws --profile sensorsyn-mfa rds describe-db-instances \
  --db-instance-identifier odoo-production \
  --query "DBInstances[0].{StorageType:StorageType,IOPS:Iops,Status:DBInstanceStatus}" \
  --output table --region ap-southeast-2
```

**Pass criteria:**
- `StorageType` = `gp3`
- `DBInstanceStatus` = `available`
- `RDS CPU`, `RDS free memory`, `RDS connections` in health-check all ✅ PASS
- No increase in ALB 5xx error rate
- Check Odoo is accessible and functional

---

## Rollback

If application errors spike or RDS shows unhealthy metrics, revert immediately:

```bash
aws --profile sensorsyn-mfa rds modify-db-instance \
  --db-instance-identifier odoo-production \
  --storage-type io1 \
  --iops 3000 \
  --apply-immediately \
  --region ap-southeast-2
```

**Note:** Reverting from gp3 back to io1 is uncommon and will re-incur the $330/month PIOPS
cost, but data is safe throughout.

---

## Monitor

Check RDS CloudWatch metrics (CPU, FreeableMemory, ReadIOPS, WriteIOPS) for 48 hours after
the change. Cost reduction will appear in the next AWS billing cycle.
