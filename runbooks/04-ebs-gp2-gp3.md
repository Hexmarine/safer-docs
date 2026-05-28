# Runbook 04 — EC2 EBS Volume Migration: gp2 → gp3

**Proposal 5** from `docs/investigations/2026-04-11-cost-optimization-proposals.md`

- **Expected saving:** ~$46/month (gp3 is ~20% cheaper than gp2 per GB)
- **Risk:** Very low — online operation, no data movement, no downtime
- **Compliance:** ✅ Storage type change does not affect data or retention
- **Approval required:** Written approval

---

## Context

55 in-use EC2 EBS volumes totalling 2,435 GB are on gp2 (as of April 2026).
- gp2: $0.114/GB-month = ~$277/month
- gp3: $0.096/GB-month = ~$234/month for same capacity
- gp3 also includes 3,000 IOPS and 125 MB/s baseline at no extra charge (vs gp2's burst model)
- Migration is performed live by AWS; no restart or downtime required

**Note:** Do not modify RDS volumes (RDS io1→gp3 is covered in runbook 02). This runbook
covers EC2 instance volumes only.

---

## Pre-change

```bash
bash scripts/health-check.sh --save-baseline tmp/baseline-ebs-gp2-gp3-$(date +%Y%m%d-%H%M%S).json
```

Get the list of in-use gp2 volumes attached to running instances:
```bash
aws --profile sensorsyn-mfa ec2 describe-volumes \
  --filters "Name=volume-type,Values=gp2" "Name=status,Values=in-use" \
  --query "Volumes[*].{ID:VolumeId,Size:Size,Instance:Attachments[0].InstanceId}" \
  --output table --region ap-southeast-2
```

---

## Approval required

- [ ] Written approval for migrating all in-use gp2 volumes to gp3

---

## Change commands

**A human must run these after approval. Can be batched — all are safe to run concurrently.**

```bash
# Get all in-use gp2 volume IDs
VOLUMES=$(aws --profile sensorsyn-mfa ec2 describe-volumes \
  --filters "Name=volume-type,Values=gp2" "Name=status,Values=in-use" \
  --query "Volumes[*].VolumeId" --output text --region ap-southeast-2)

echo "Volumes to migrate: $(echo $VOLUMES | wc -w)"

# Migrate each volume to gp3 (can run in parallel — no downtime)
for VOL in $VOLUMES; do
  aws --profile sensorsyn-mfa ec2 modify-volume \
    --volume-id "$VOL" --volume-type gp3 --region ap-southeast-2
  echo "Modifying $VOL → gp3"
done
```

Migration typically completes in a few minutes per volume. Monitor progress:
```bash
aws --profile sensorsyn-mfa ec2 describe-volumes-modifications \
  --query "VolumesModifications[?OriginalVolumeType=='gp2'].{ID:VolumeId,State:ModificationState,Progress:Progress}" \
  --output table --region ap-southeast-2
```

Expected progression: `modifying` → `optimizing` → `completed`

---

## Verification

```bash
bash scripts/health-check.sh --compare-to tmp/baseline-ebs-gp2-gp3-*.json
```

Confirm all volumes are now gp3:
```bash
aws --profile sensorsyn-mfa ec2 describe-volumes \
  --filters "Name=volume-type,Values=gp2" "Name=status,Values=in-use" \
  --query "length(Volumes)" --output text --region ap-southeast-2
# Should return 0
```

**Pass criteria:**
- All health metrics unchanged ✅
- Zero remaining in-use gp2 volumes
- No EC2 instance errors or CloudWatch alarms triggered

---

## Rollback

Revert a volume back to gp2 if needed:
```bash
aws --profile sensorsyn-mfa ec2 modify-volume \
  --volume-id vol-XXXXXXXXXXXXXXXXX --volume-type gp2 --region ap-southeast-2
```

In practice, rollback is very unlikely — gp3 performs the same as gp2 for virtually all
workloads, and the migration is non-disruptive.
