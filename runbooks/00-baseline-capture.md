# Runbook 00 — Pre-Change Baseline Capture

Run this before applying **any** cost optimization change. Takes ~2 minutes.

## Prerequisites

- [ ] MFA session is active (`aws --profile sensorsyn-mfa sts get-caller-identity` succeeds)
- [ ] No active incidents or on-call alerts for the platform
- [ ] Confirm the change window is agreed (for production changes)

## Baseline capture

```bash
# Replace CHANGE-NAME with a short identifier, e.g. "ebs-dlm", "rds-io1-gp3", "log-retention"
bash scripts/health-check.sh --save-baseline tmp/baseline-CHANGE-NAME-$(date +%Y%m%d-%H%M%S).json
```

Expected output: a table showing 10 platform metrics, all either ✅ PASS or ℹ️ INFO.

**Do not proceed if any metric shows ❌ FAIL.** Investigate the failure first.

⚠️ metrics (WARN — metric not available) usually mean the MFA session is expired. Refresh it
with `bash scripts/aws-mfa-login.sh` and re-run.

## When to re-run verification

After applying the change, run:

```bash
bash scripts/health-check.sh --compare-to tmp/baseline-CHANGE-NAME-*.json
```

Timing:
- Immediately after the change
- 15 minutes after
- 1 hour after (for production storage/RDS changes)

## Baseline file management

Baseline files are saved to `tmp/` which is not committed. Keep them on your local machine
for the duration of the change window. Once the change is confirmed stable, you can delete them.

Naming convention:
```
tmp/baseline-<change>-<YYYYMMDD-HHMMSS>.json
```

Examples:
```
tmp/baseline-ebs-dlm-20260415-103000.json
tmp/baseline-rds-io1-gp3-20260416-020000.json
tmp/baseline-log-retention-20260415-143000.json
```
