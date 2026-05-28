# Runbook 03 — CloudWatch Log Group Retention Policies

**Proposal 4** from `docs/investigations/2026-04-11-cost-optimization-proposals.md`

- **Expected saving:** $50–200/month (stops unbounded log accumulation in non-prod groups)
- **Risk:** Very low — logs older than the retention period are deleted; no live data affected
- **Compliance:** ✅ Non-prod: no mandate. Prod: NEVER_EXPIRE is fine, but 365 days is sufficient.
- **Approval required:** Written approval

---

## Context

All 143 CloudWatch log groups have `NEVER_EXPIRE` retention. Non-prod VPC flow logs (qa, dev,
uat, sandbox) are 35–50 GB each and growing. Prod logs must stay at 365 days (SOC 2).

This first automation rollout is intentionally **non-prod first** and uses a **hardcoded allowlist**
in `scripts/set-log-retention.sh`. The script now also supports a separate explicit prod scope for
known 365-day groups, but shared `RDSOSMetrics` and pattern-only `sso-*` groups still stay out of
automation until a fresh live inventory is pulled and approved.

The script now supports two non-prod policy levels:
- **Standard / safer default:** 30 days infra/security, 90 days app/error
- **Aggressive minimum:** 7 days infra/security, 14 days app/error
- **Separate prod scopes:** 365/90/30/14/7 day options for known prod/WAF groups via `prod-*` / `prd-*`

**Policy to apply in this first rollout:**

| Log group | New retention | Rationale |
|---|---|---|
| `vpc-flow-log-development` | 30 days | Non-prod VPC flow — no compliance requirement |
| `qa-vpc-flow-logs` | 30 days | Non-prod VPC flow |
| `sandbox-vpc-flow-log` | 30 days | Non-prod VPC flow |
| `vpc-uat-log-group-name` | 30 days | Non-prod VPC flow |
| `aws-waf-logs-sandbox` | 30 days | Non-prod security / WAF log |
| `smoke-qa-api-out` | 90 days | Non-prod app log |
| `smoke-dev-api-out` | 90 days | Non-prod app log |
| `smoke-api-sandbox-pm2-out-log` | 90 days | Non-prod app log |
| `smoke-qa-api-error` | 90 days | Non-prod app error log |
| `smoke-dev-api-error` | 90 days | Non-prod app error log |

**Aggressive minimum alternative:**
- Set the 30-day group set above to **7 days**
- Set the 90-day group set above to **14 days**
- Use only if the team explicitly accepts a shorter debugging window

**Separate prod/compliance scopes available in the script:**
- `smoke-api-prod-pm2-out-log`
- `smoke-vpc-flow-log`
- `aws-waf-logs-prod`
- Available retention options: **365 / 90 / 30 / 14 / 7 days**
- **365 days remains the compliance-aligned default**
- Use shorter prod windows only as a separately-approved exception

---

## Pre-change

```bash
bash scripts/health-check.sh --save-baseline tmp/baseline-log-retention-$(date +%Y%m%d-%H%M%S).json
```

Also capture current log group sizes for the non-prod groups:
```bash
aws --profile sensorsyn-mfa logs describe-log-groups \
  --query "logGroups[?contains(logGroupName,'dev')||contains(logGroupName,'qa')||contains(logGroupName,'sandbox')||contains(logGroupName,'uat')].{Name:logGroupName,MB:storedBytes,Retention:retentionInDays}" \
  --output table --region ap-southeast-2
```

Optional: show the exact scripted target list before running anything:
```bash
bash scripts/set-log-retention.sh --list-targets
```

---

## Approval required

- [ ] Written approval for setting non-prod log groups to either 30–90 days or the aggressive 7–14 day minimum
- [ ] Confirm with the team that no non-prod debugging sessions rely on logs older than the selected window
- [ ] If using any `prod-*` / `prd-*` scope, separate written approval exists for the prod step
- [ ] If using prod retention shorter than 365 days, business/compliance sign-off exists for that exception

---

## Change commands

**A human must run these after approval.**

Preview the VPC/security scope:
```bash
bash scripts/set-log-retention.sh --scope nonprod-30d --dry-run
```

Preview the non-prod app scope:
```bash
bash scripts/set-log-retention.sh --scope nonprod-90d --dry-run
```

If using the aggressive minimum policy instead:
```bash
bash scripts/set-log-retention.sh --scope nonprod-7d --dry-run
bash scripts/set-log-retention.sh --scope nonprod-14d --dry-run
```

Preview the prod scope separately:
```bash
bash scripts/set-log-retention.sh --scope prd-365d --dry-run
bash scripts/set-log-retention.sh --scope prd-90d --dry-run
bash scripts/set-log-retention.sh --scope prd-30d --dry-run
bash scripts/set-log-retention.sh --scope prd-14d --dry-run
bash scripts/set-log-retention.sh --scope prd-7d --dry-run
```

Execute the 30-day scope:
```bash
bash scripts/set-log-retention.sh --scope nonprod-30d
```

Execute the 90-day scope:
```bash
bash scripts/set-log-retention.sh --scope nonprod-90d
```

Execute the aggressive minimum policy instead:
```bash
bash scripts/set-log-retention.sh --scope nonprod-7d
bash scripts/set-log-retention.sh --scope nonprod-14d
```

Execute the prod scope separately:
```bash
bash scripts/set-log-retention.sh --scope prd-365d
bash scripts/set-log-retention.sh --scope prd-90d
bash scripts/set-log-retention.sh --scope prd-30d
bash scripts/set-log-retention.sh --scope prd-14d
bash scripts/set-log-retention.sh --scope prd-7d
```

---

## Verification

```bash
bash scripts/health-check.sh --compare-to tmp/baseline-log-retention-*.json
```

Confirm retention policies were applied:
```bash
aws --profile sensorsyn-mfa logs describe-log-groups \
  --query "logGroups[?contains(logGroupName,'dev')||contains(logGroupName,'qa')||contains(logGroupName,'sandbox')||contains(logGroupName,'uat')].{Name:logGroupName,Retention:retentionInDays}" \
  --output table --region ap-southeast-2
```

**Pass criteria:**
- All health metrics unchanged ✅
- Standard rollout: 30-day non-prod groups show `retentionInDays: 30`
- Standard rollout: 90-day non-prod groups show `retentionInDays: 90`
- Aggressive minimum rollout: infra/security groups show `retentionInDays: 7`
- Aggressive minimum rollout: app/error groups show `retentionInDays: 14`
- If prod scope executed: known prod groups show the explicitly-selected retention (`365/90/30/14/7`)
- Shared `RDSOSMetrics` and undocumented `sso-*` groups remain unchanged

---

## Rollback

Restore any affected group to no-expire:
```bash
aws --profile sensorsyn-mfa logs delete-retention-policy \
  --log-group-name GROUP_NAME --region ap-southeast-2
```

This restores `NEVER_EXPIRE` for that group. Data already deleted by the retention policy
cannot be recovered, but since we set 30/90-day policies and CloudWatch deletes expired
logs on its own schedule (not immediately), the window for this is narrow.

---

## Notes

- Log deletion by CloudWatch happens progressively after the retention policy is set —
  it does not delete all historical logs instantly.
- Cost savings will appear gradually over the following 30–90 days as old data ages out.
- The biggest immediate wins are the non-prod VPC flow log groups (35–50 GB each).
- The 7/14-day mode is the bare minimum operational posture, not the recommended default.
- Shared `RDSOSMetrics` and exact `sso-*` groups are still deferred until MFA is refreshed and the
  live inventory is re-checked.
