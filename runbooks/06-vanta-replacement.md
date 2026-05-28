# Runbook 06 — Vanta → AWS Audit Manager Transition

**Proposal 15** from `docs/investigations/2026-04-11-cost-optimization-proposals.md`

- **Expected saving:** ~$940–1,010/month (eliminating ~$1,042/month Vanta cost)
- **Risk:** Low if planned carefully; high if rushed mid-audit
- **Compliance:** ✅ SOC 2 remains achievable with AWS Audit Manager; evidence gap must be closed first
- **Approval required:** Business decision — requires contract owner sign-off and audit schedule review

---

## Context

Vanta costs ~$12,500/year (inferred from billing history). The account already has:
- AWS Security Hub, Config, Inspector, GuardDuty, Macie — all running
- 290 CloudWatch alarms — continuous monitoring in place

The only functional gap is **SOC 2 evidence packaging and reporting**, which AWS Audit Manager
provides at ~$30–100/month (vs ~$1,042/month equivalent for Vanta).

**This is a planning-heavy, multi-week transition — not a single CLI operation.**

---

## Phase 1 — Gather information (prerequisite — no AWS changes)

Before any action:

1. **Get the Vanta renewal date** — contact the contract owner. Typical notice period is 30 days.
   Do not cancel without confirming the renewal window.

2. **Confirm no active SOC 2 audit is in progress** — Vanta removal mid-audit would invalidate
   evidence collection. Only proceed if the next audit is ≥60 days away.

3. **Check Vanta trust center usage** — If customers access the Vanta trust center URL to view
   your compliance status, this must be replaced before cancellation.
   Options: AWS Artifact (internal only), a static trust page, or a third-party trust center.

4. **Inventory Vanta-managed integrations** — List all third-party systems connected to Vanta
   (e.g., GitHub, Jira, HR system). These need equivalent coverage in Audit Manager or Config.

---

## Phase 2 — Enable AWS Audit Manager (read-only evaluation first)

**These are setup commands. They do not affect production workloads.**

```bash
# Check if Audit Manager is already enabled
aws --profile sensorsyn-mfa auditmanager get-account-status --region ap-southeast-2

# If not enabled, enable it (one-time setup — no production impact)
aws --profile sensorsyn-mfa auditmanager register-account --region ap-southeast-2
```

Once enabled, create a SOC 2 assessment framework:
```bash
# List available frameworks (look for SOC 2)
aws --profile sensorsyn-mfa auditmanager list-assessment-frameworks \
  --framework-type Standard --region ap-southeast-2 | python3 -c "
import json,sys
frameworks=json.load(sys.stdin)['frameworkMetadataList']
for f in frameworks:
    if 'SOC' in f.get('name','').upper():
        print(f['id'], f['name'])
"
```

```bash
# Create a SOC 2 assessment (replace FRAMEWORK_ID from the list above)
aws --profile sensorsyn-mfa auditmanager create-assessment \
  --name "SOC2-2026" \
  --assessment-reports-destination '{"destinationType":"S3","destination":"s3://sensor-global-cloudtrail-logs/audit-manager-reports/"}' \
  --scope '{"awsAccounts":[{"id":"747293622182"}],"awsServices":[{"serviceName":"EC2"},{"serviceName":"RDS"},{"serviceName":"S3"},{"serviceName":"IAM"},{"serviceName":"CloudTrail"}]}' \
  --framework-id FRAMEWORK_ID \
  --roles '[{"roleArn":"arn:aws:iam::747293622182:role/AWSAuditManagerDefaultRole","roleType":"PROCESS_OWNER"}]' \
  --region ap-southeast-2
```

---

## Phase 3 — Run Audit Manager in parallel (30-day validation period)

Run Audit Manager alongside Vanta for at least 30 days before cancelling Vanta. This ensures:
- Evidence collection is working
- SOC 2 control coverage is equivalent
- The team is comfortable with Audit Manager workflows

Check evidence collection status:
```bash
aws --profile sensorsyn-mfa auditmanager list-assessments --region ap-southeast-2
```

---

## Phase 4 — Cancel Vanta at renewal

Once Audit Manager is validated and the renewal date is approaching:
1. Give 30 days notice to Vanta before the renewal date
2. Export all historical evidence from Vanta before account closure
3. Confirm any active auditors have been transitioned to Audit Manager reports

---

## Health check (no pre/post comparison needed — this is a planning change)

After enabling Audit Manager:
```bash
bash scripts/health-check.sh
```

Audit Manager should have zero impact on platform health metrics. Any unexpected alarms
in the health check after enabling Audit Manager are coincidental and should be investigated
independently.

---

## Cost tracking

After cancelling Vanta:
- AWS Audit Manager cost will appear in the AWS bill under `AWS Audit Manager`
- Expected: $30–100/month depending on resource count
- Verify in Cost Explorer 30 days after enabling

---

## Known gaps (accept or mitigate)

| Gap | Mitigation |
|---|---|
| Public trust center | Create a static trust page on the marketing site, or use a cheaper trust center tool |
| Vendor risk management | Internal process + manual reviews; document in Confluence/Notion |
| Questionnaire automation | Manual; typically <4 questionnaires/year for a company this size |
| Employee policy acknowledgement tracking | HR system or a simple Google Form |
