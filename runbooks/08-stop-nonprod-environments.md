# Runbook 08 — Stop / Resume Non-Production Environments

**Applies to:** All non-prod environments (dev, qa, sandbox, and optionally UAT)  
**Risk:** Low — production is untouched; only non-prod resources affected  
**Approval required:** Yes — written sign-off from engineering lead before first execution  
**Estimated saving:** ~$620–700/month (EC2 + RDS compute; ElastiCache not included)

---

## Overview

This runbook describes how to shut down all non-production AWS environments in one command
and bring them back on-demand. It uses two scripts:

| Script | Purpose |
|---|---|
| `scripts/stop-nonprod.sh` | Saves state, zeros non-prod ASGs, stops non-prod RDS |
| `scripts/start-nonprod.sh` | Reads state file, starts RDS, restores ASG counts |

**What is stopped:**

| Resource | Dev | QA | Sandbox | UAT (opt-in) | Prod |
|---|---|---|---|---|---|
| EC2 via ASG | ✅ | ✅ | ✅ | with `--include-uat` | ⛔ never |
| RDS | ✅ | ✅ | ✅ | with `--include-uat` | ⛔ never |
| ElastiCache | ❌ cannot stop | ❌ | ❌ | ❌ | ⛔ never |

**ElastiCache caveat:** AWS does not support stopping Redis clusters without deleting them.
Dev/QA clusters are t3.micro (~$20/node-month) and remain running. See Proposal 9 in
`docs/investigations/2026-04-11-cost-optimization-proposals.md` for scheduled off-hours automation.

---

## Step 0 — Prerequisites

Before stopping anything:

- [ ] Verify that no active development sprint is in progress that requires non-prod
- [ ] Verify that no UAT session / customer demo is scheduled in the next 24 hours
- [ ] Confirm with team whether UAT should be included (`--include-uat`)
- [ ] Run a health baseline (optional but recommended):
  ```bash
  bash scripts/health-check.sh --save-baseline tmp/baseline-before-nonprod-stop-$(date +%Y%m%d).json
  ```
- [ ] Refresh MFA credentials if needed:
  ```bash
  bash scripts/aws-mfa-login.sh default sensorsyn-mfa
  ```

---

## Step 1 — Stop Non-Prod Environments

### Option A: Stop dev, qa, sandbox only (default — UAT stays up)

```bash
bash scripts/stop-nonprod.sh --dry-run    # preview what will happen
bash scripts/stop-nonprod.sh              # execute
```

### Option B: Stop everything including UAT

```bash
bash scripts/stop-nonprod.sh --dry-run --include-uat
bash scripts/stop-nonprod.sh --include-uat
```

**The script will:**
1. Verify you're in account `747293622182` (aborts otherwise)
2. Save current ASG min/desired/max counts to `tmp/nonprod-state-TIMESTAMP.json`
3. Set each non-prod ASG to `min=0, desired=0` (instances terminate gracefully)
4. Call `aws rds stop-db-instance` for each non-prod RDS instance

**Note the state file path** printed at the end — you need it to resume.

---

## Step 2 — Verify Environments Are Down

Wait 3–5 minutes, then verify:

```bash
# Check ASG instance counts (should all be 0 for non-prod)
aws --profile sensorsyn-mfa autoscaling describe-auto-scaling-groups \
  --query 'AutoScalingGroups[?DesiredCapacity>`0`].{Name:AutoScalingGroupName,Desired:DesiredCapacity}' \
  --output table --region ap-southeast-2

# Check RDS instance statuses (non-prod should show "stopping" or "stopped")
aws --profile sensorsyn-mfa rds describe-db-instances \
  --query 'DBInstances[*].{ID:DBInstanceIdentifier,Status:DBInstanceStatus}' \
  --output table --region ap-southeast-2
```

**Expected result:** Only `CodeDeploy_Smoke-API_d-NNRF41DGD` (prod) and `CodeDeploy_sso-api-dg_d-QKLSZP50D` (prod SSO) should have non-zero desired capacity. All others should be 0.

---

## Step 3 — Keeping Non-Prod Down Long-Term

> ⚠️ **AWS automatically restarts stopped RDS instances after 7 days.** This is a hard AWS limit.

If you want non-prod environments to stay down for more than 7 days, you must re-run the stop
script weekly:

```bash
# Re-run weekly to prevent AWS from auto-restarting RDS
bash scripts/stop-nonprod.sh [--include-uat]
```

### Optional: Automate the weekly re-stop with EventBridge

To avoid manual re-running, create an EventBridge Scheduler rule that calls an SSM Automation
document (or Lambda) running the stop commands on a weekly schedule. This is not in scope for
this runbook but is the recommended approach for long-term cost reduction.

Command to add to automation:
```
# For each non-prod RDS instance:
aws rds stop-db-instance --db-instance-identifier <INSTANCE_ID>
```

---

## Step 4 — Resume Environments

When the team needs non-prod back:

```bash
# List available state files
ls -lt tmp/nonprod-state-*.json

# Resume from the most recent stop
bash scripts/start-nonprod.sh \
  --state-file tmp/nonprod-state-TIMESTAMP.json

# If UAT was also stopped:
bash scripts/start-nonprod.sh \
  --state-file tmp/nonprod-state-TIMESTAMP.json \
  --include-uat
```

**The script will:**
1. Start each non-prod RDS instance
2. Wait up to 10 minutes for each to reach `available` state (use `--no-wait` to skip waiting)
3. Restore each non-prod ASG to its saved min/desired/max

**After resuming — verify health:**
```bash
bash scripts/health-check.sh
```

---

## Rollback

If something goes wrong during stop (e.g., an ASG is partially zeroed):

```bash
# Resume immediately using the state file that was saved before stopping
bash scripts/start-nonprod.sh --state-file tmp/nonprod-state-TIMESTAMP.json [--no-wait]
```

If no state file is available, manually restore ASG counts from the table in plan.md:

| ASG | Min | Desired | Max |
|---|---|---|---|
| CodeDeploy_Smokealarmdevelopmentapi_d-KHX1BJY5D | 2 | 2 | 4 |
| CodeDeploy_dev-sso-api-dg_d-4O83LCN0D | 1 | 1 | 2 |
| CodeDeploy_Smoke-qa-API_d-NK1QJ9K51 | 3 | 3 | 4 |
| CodeDeploy_qa-sso-api-dg_d-Y5R4U1N0D | 1 | 1 | 2 |
| CodeDeploy_Smoke-Sandbox-Api-Dg_d-EMHG7U561 | 2 | 2 | 3 |
| CodeDeploy_sandbox-sso-api-dg_d-P0SQWI50D | 1 | 1 | 2 |
| CodeDeploy_uat-api-deployment-group_d-KNG5N8TB8 | 1 | 1 | 1 |
| CodeDeploy_uat-sso-dg_d-IB9IAZOB8 | 1 | 1 | 1 |

---

## Approval Gate

**Written approval required from engineering lead before first execution.**

Include in the approval:
- Confirmation of the date range non-prod will be down
- Whether UAT should be included (`--include-uat`)
- Confirmation that no active sprint or customer demo will be affected
