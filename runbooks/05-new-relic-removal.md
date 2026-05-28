# Runbook 05 — Remove New Relic Metric Stream

**Proposal 16** from `docs/investigations/2026-04-11-cost-optimization-proposals.md`

- **Expected saving:** ~$756/month (MetricStreamUsage + Firehose processing eliminated)
- **Risk:** Medium — New Relic dashboards/alerts will lose data; must migrate first
- **Compliance:** ✅ Monitoring changes have no compliance impact; 290 CloudWatch alarms unaffected
- **Approval required:** Written approval + confirmation that NR dashboards have been migrated

---

## Context

`NewRelic-Metric-Stream` was created 2023-05-09 and streams **all** CloudWatch namespaces to
New Relic via Kinesis Firehose. Monthly cost breakdown:
- CloudWatch metric stream usage: ~$579/month
- Kinesis Firehose processing: ~$172/month + ~$5/month
- **Total: ~$756/month**

The 290 existing CloudWatch alarms do NOT depend on the metric stream — they query CloudWatch
metrics directly. Deleting the stream does not affect alerting.

**This is a two-phase change:**
1. **Phase A (prerequisite):** Audit New Relic dashboards, recreate critical ones in CloudWatch
2. **Phase B (this runbook):** Stop the metric stream and Firehose delivery stream

---

## Phase A — New Relic dashboard audit (prerequisite)

Before stopping the stream:

1. List all New Relic dashboards that were recently accessed (last 30 days)
2. For each critical dashboard, identify which CloudWatch metrics it queries
3. Recreate critical dashboards in CloudWatch Dashboards ($3/dashboard/month)
4. Get sign-off from each dashboard owner that their CloudWatch equivalent is sufficient

**This is an engineering task, not a CLI operation.** It cannot be captured as a script.
Estimated effort: 1–2 days for a typical set of 5–10 operational dashboards.

---

## Pre-change (Phase B)

```bash
bash scripts/health-check.sh --save-baseline tmp/baseline-new-relic-removal-$(date +%Y%m%d-%H%M%S).json
```

Confirm the stream exists and its current state:
```bash
aws --profile sensorsyn-mfa cloudwatch get-metric-stream \
  --name NewRelic-Metric-Stream --region ap-southeast-2

aws --profile sensorsyn-mfa firehose describe-delivery-stream \
  --delivery-stream-name NewRelic-Delivery-Stream --region ap-southeast-2
```

---

## Approval required

- [ ] Written approval from the New Relic integration owner
- [ ] Confirmation that all critical NR dashboards have been replicated in CloudWatch
- [ ] Sign-off from account owner for stream deletion

---

## Phase B — Stop and delete the stream

**A human must run these after approval.**

Step 1: Stop the metric stream (pauses but does not delete — can be restarted if needed):
```bash
aws --profile sensorsyn-mfa cloudwatch stop-metric-streams \
  --names NewRelic-Metric-Stream --region ap-southeast-2
```

Wait 10 minutes and verify no alerts or issues are reported.

Step 2: Delete the metric stream:
```bash
aws --profile sensorsyn-mfa cloudwatch delete-metric-stream \
  --name NewRelic-Metric-Stream --region ap-southeast-2
```

Step 3: Delete the Firehose delivery stream:
```bash
aws --profile sensorsyn-mfa firehose delete-delivery-stream \
  --delivery-stream-name NewRelic-Delivery-Stream --region ap-southeast-2
```

---

## Verification

```bash
bash scripts/health-check.sh --compare-to tmp/baseline-new-relic-removal-*.json
```

Confirm streams are gone:
```bash
aws --profile sensorsyn-mfa cloudwatch list-metric-streams --region ap-southeast-2
aws --profile sensorsyn-mfa firehose list-delivery-streams --region ap-southeast-2
```

**Pass criteria:**
- All health metrics unchanged ✅
- CloudWatch alarms still functioning (check a sample alarm's state)
- No new ERROR-level entries in CloudWatch Logs from application services
- New Relic-derived dashboards replaced and confirmed working

---

## Rollback

Recreate the metric stream (restores all-namespace streaming):
```bash
aws --profile sensorsyn-mfa cloudwatch put-metric-stream \
  --name NewRelic-Metric-Stream \
  --output-arn <original-firehose-arn> \
  --output-format JSON \
  --region ap-southeast-2
```

**Note:** The Firehose delivery stream must exist before the metric stream can be recreated.
If the Firehose was also deleted, it must be recreated first (requires New Relic destination
configuration including the NR API key). For this reason, keep the New Relic API key in
Secrets Manager until the change is confirmed stable for 30 days.

---

## Cost validation

Check CloudWatch cost in AWS Cost Explorer 7 days after deletion:
- `MetricStreamUsage` line should drop to zero
- `DataProcessing-Bytes` associated with Firehose should drop significantly
