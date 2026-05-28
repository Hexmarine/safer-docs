# Runbook 01 — EBS Snapshot Lifecycle Policy (DLM)

**Proposal 1** from `docs/investigations/2026-04-11-cost-optimization-proposals.md`

- **Expected saving:** $350–500/month (plus ongoing cost growth prevention)
- **Risk:** Low — new snapshots continue to be created; old ones are deleted on a schedule
- **Compliance:** ✅ SOC 2 minimum is 90 days. Team confirmed 90-day retention is acceptable.
- **Approval required:** Written approval in this conversation or linked GitHub issue

---

## Context

The account currently has 5,064+ EBS snapshots (April 2026), 93.7% older than 90 days. No DLM
policy exists. Snapshot cost has grown from $387/month (July 2025) to $631/month (April 2026).

**This is a two-step change:**
1. Create a DLM lifecycle policy (new policy — no existing data deleted immediately)
2. Bulk delete snapshots older than 90 days (manual one-time cleanup — requires separate approval)

---

## Pre-change

```bash
bash scripts/health-check.sh --save-baseline tmp/baseline-ebs-dlm-$(date +%Y%m%d-%H%M%S).json
```

Also record the current snapshot count:
```bash
aws --profile sensorsyn-mfa ec2 describe-snapshots --owner-ids self \
  --query "length(Snapshots)" --output text
```

---

## Approval required

- [ ] Written approval from account owner for the DLM policy creation
- [ ] Separate written approval before bulk deletion of snapshots older than 90 days

---

## Step 1 — Create DLM lifecycle policy

**These commands are for reference. A human must run them after approval.**

```bash
# Create a role for DLM (first time only — check if it already exists)
aws --profile sensorsyn-mfa iam get-role --role-name AWSDataLifecycleManagerDefaultRole

# Create the lifecycle policy
aws --profile sensorsyn-mfa dlm create-lifecycle-policy \
  --description "90-day EBS snapshot retention — all volumes ap-southeast-2" \
  --state ENABLED \
  --execution-role-arn arn:aws:iam::747293622182:role/AWSDataLifecycleManagerDefaultRole \
  --policy-details '{
    "PolicyType": "EBS_SNAPSHOT_MANAGEMENT",
    "ResourceTypes": ["VOLUME"],
    "TargetTags": [{"Key": "backup", "Value": "true"}],
    "Schedules": [{
      "Name": "90-day-retention",
      "CreateRule": {"Interval": 24, "IntervalUnit": "HOURS", "Times": ["02:00"]},
      "RetainRule": {"Count": 90},
      "CopyTags": true
    }]
  }' \
  --region ap-southeast-2
```

**Note:** The DLM policy uses a tag filter (`backup=true`). Before running, confirm which volumes
are currently being snapshotted manually and whether they carry this tag. If volumes don't have
the tag, the target filter should be adjusted to match the existing pattern.

---

## Step 2 — Bulk delete old snapshots (separate approval)

**Do not run this until the DLM policy has been confirmed active for at least 24 hours.**

```bash
# Preview: list snapshots older than 90 days (read-only)
aws --profile sensorsyn-mfa ec2 describe-snapshots --owner-ids self \
  --query "Snapshots[?StartTime<='2026-01-11'].{ID:SnapshotId,Date:StartTime,Size:VolumeSize}" \
  --output table | head -50
```

The bulk delete should be done via a script with a dry-run mode first. Get approval before
running the deletion script. Target: all snapshots with `StartTime` older than 90 days.

---

## Verification

```bash
bash scripts/health-check.sh --compare-to tmp/baseline-ebs-dlm-*.json
```

Also check the snapshot count is not dramatically lower (DLM creates, not deletes, in step 1):
```bash
aws --profile sensorsyn-mfa ec2 describe-snapshots --owner-ids self \
  --query "length(Snapshots)" --output text
```

**Pass criteria:**
- All health metrics remain ✅ PASS or ℹ️ INFO
- No application errors spike after bulk deletion (check ALB 5xx rate)
- Snapshot count decreases over the following billing cycle (confirming DLM is deleting old ones)

---

## Rollback

**DLM policy:** Disable or delete the policy via console or:
```bash
# First get the policy ID
aws --profile sensorsyn-mfa dlm get-lifecycle-policies --region ap-southeast-2

# Disable it
aws --profile sensorsyn-mfa dlm update-lifecycle-policy --policy-id <ID> --state DISABLED --region ap-southeast-2
```

**Bulk-deleted snapshots:** Cannot be recovered — this is why bulk deletion requires separate
approval and should only happen after DLM is confirmed working.

---

## Monitor

Check snapshot cost in AWS Cost Explorer 7 days after the bulk deletion. It should begin
decreasing immediately. Full cost impact will be visible after 30 days.
