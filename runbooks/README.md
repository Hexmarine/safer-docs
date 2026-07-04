# Runbooks — Cost Optimization Change Verification Harness

This directory contains step-by-step runbooks for each cost optimization proposal. Each runbook
defines the pre-change baseline capture, the change itself (approval-gated), post-change
verification, explicit pass/fail criteria, and rollback instructions.

## How to use

### Before any change

```bash
# 1. Make sure your MFA session is live
bash scripts/aws-mfa-login.sh

# 2. Capture a baseline (saves to tmp/)
bash scripts/health-check.sh --save-baseline tmp/baseline-CHANGE-NAME-$(date +%Y%m%d-%H%M%S).json
```

The baseline captures 10 platform health metrics:

| Metric | What it watches |
|---|---|
| ALB healthy hosts | Are prod API servers responding behind the load balancer? |
| ALB 5xx error rate | Is the API returning errors? |
| RDS CPU (odoo-production) | Is the database under pressure? |
| RDS free memory | Is the database under memory pressure? |
| RDS free storage | Is the disk filling up? |
| RDS connections | Is the database accepting queries? |
| EC2 instances ok/total | Are EMQX + prod API EC2 instances healthy? |
| CW alarms firing | How many alarms are currently in ALARM state? |
| SMS sends (last 24h) | Are smoke alarm notifications flowing? (safety check) |
| EBS snapshot count | Tracks snapshot DLM progress over time |

### After applying a change

```bash
# Compare current state against the saved baseline
bash scripts/health-check.sh --compare-to tmp/baseline-CHANGE-NAME-*.json
```

The comparison flags any metric that regressed significantly (>10% on "lower is worse" metrics,
>20% on "higher is worse" metrics).

### Pass criteria
- All FAIL indicators resolve to PASS
- No new ⚠️ regressions in the comparison vs baseline
- Wait 15 minutes, then re-run the comparison

### Rollback trigger
If any FAIL or ⚠️ regression appears and cannot be explained, follow the rollback steps in
the relevant runbook immediately. Do not wait.

---

## Runbooks index

| File | Proposal | Est. saving | Risk |
|---|---|---|---|
| [00-baseline-capture.md](00-baseline-capture.md) | Pre-change checklist | — | — |
| [01-ebs-snapshot-dlm.md](01-ebs-snapshot-dlm.md) | EBS snapshot lifecycle policy | $350–500/month | Low |
| [02-rds-io1-gp3.md](02-rds-io1-gp3.md) | odoo-production io1 → gp3 | ~$330/month | Low (prod change) |
| [03-log-retention.md](03-log-retention.md) | CloudWatch log retention | $50–200/month | Very low |
| [04-ebs-gp2-gp3.md](04-ebs-gp2-gp3.md) | EC2 gp2 → gp3 volume migration | ~$46/month | Very low |
| [05-new-relic-removal.md](05-new-relic-removal.md) | Remove New Relic metric stream | ~$756/month | Medium |
| [06-vanta-replacement.md](06-vanta-replacement.md) | Vanta → AWS Audit Manager | ~$940–1,010/month | Low (planning heavy) |
| [07-local-db-access.md](07-local-db-access.md) | Local investigation access to MySQL, MongoDB, Redis via SSM port forwarding | — | Read-only |
| [08-stop-nonprod-environments.md](08-stop-nonprod-environments.md) | Stop/resume non-prod environments | ~A$1,000/month before UAT/QA/Dev decommissioning | Low |
| [09-aws-account-standalone-detach.md](09-aws-account-standalone-detach.md) | Detach AWS account `747293622182` from parent org and operate standalone under JV billing | — | Medium–High |
| [11-mongo-custody-and-aws-restore.md](11-mongo-custody-and-aws-restore.md) | AWS-controlled MongoDB custody dump, JV Atlas restore, and DocumentDB fallback pilot | - | Medium-High |
| [12-aws-access-containment-follow-up.md](12-aws-access-containment-follow-up.md) | Post-incident AWS IAM access cleanup after Appinventiv/OneLogin containment | - | High |
| [13-ses-sensorglobal-domain-provisioning.md](13-ses-sensorglobal-domain-provisioning.md) | Provision SES for `sensorglobal.com` before backend email cutover | - | Medium |
| [14-deploy-mqtt-forensic-tap.md](14-deploy-mqtt-forensic-tap.md) | Deploy the independent MQTT forensic tap (`tbl_device_events` journal) | - | Low |
| [15-workstation-recovery.md](15-workstation-recovery.md) | Recover the dev workstation + Claude knowledge base (snapshot restore or rebuild) | - | Low |

For log retention changes, use `scripts/set-log-retention.sh` for the first non-prod rollout rather
than ad hoc CLI loops. The script keeps an explicit allowlist and supports `--dry-run`.
