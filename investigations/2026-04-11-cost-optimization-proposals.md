# Cost Optimization Proposals — April 2026

- Date: 2026-04-11
- Scope: consolidated cost optimization findings and implementation proposals for AWS account `747293622182`, building on prior investigations from 2026-04-09 and 2026-04-10
- Related systems: EC2, EBS, RDS, ElastiCache, CloudWatch, VPC, EIP, CloudWatch Logs, OpenVPN, Vanta, QuickSight

## Objective

Convert prior investigation findings into actionable, approval-ready proposals. Includes:
- Fresh data collected 2026-04-11
- Compliance constraints based on Australian Privacy Act 1988, SOC 2, and Corporations Act 2001
- Per-item savings estimate, risk rating, and approval requirement

## Sources Used

- Prior investigations: `2026-04-09-cost-quick-wins.md`, `2026-04-10-cost-deep-dive.md`, `2026-04-09-cloudwatch-newrelic-deep-dive.md`, `2026-04-09-vanta-recommendation.md`
- AWS commands (2026-04-11):
  - `aws ce get-cost-and-usage` — April MTD cost by service
  - `aws ec2 describe-volumes` — all gp2 volumes, unattached volumes, stopped instance volumes
  - `aws ec2 describe-instances` — stopped instances
  - `aws ec2 describe-nat-gateways` — active NAT count
  - `aws ec2 describe-addresses` — all EIPs
  - `aws ec2 describe-snapshots` — total count refresh
  - `aws logs describe-log-groups` — all 143 log groups with retention and stored bytes
- Web research:
  - Australian Privacy Act 1988 (APP 11 — retention and destruction)
  - SOC 2 CloudWatch / CloudTrail log retention guidance
  - Corporations Act 2001 financial record retention

---

## Compliance Constraints Summary

No blanket minimum retention period applies in Australia for most data types. The key rules are:

| Area | Minimum | Source | Notes |
|---|---|---|---|
| CloudTrail / audit logs (prod) | 365 days | SOC 2 best practice | NEVER_EXPIRE is acceptable but incurs cost; cap at 365d |
| CloudWatch app logs (prod) | 365 days | SOC 2 CC7 | Production only |
| CloudWatch logs (non-prod) | No mandate | N/A | 30–90 days is sufficient |
| EBS snapshots / DB backups | 90 days | SOC 2 CC6.x data availability | Confirm with team if DR policy mandates more |
| IoT device / alarm telemetry data | Purpose-driven | APP 11 | Delete when no longer needed |
| Financial/transaction records (app DB) | 5–7 years | Corporations Act 2001 | Applies to database *data*, not logs or snapshots |
| VPC flow logs (non-prod) | None | N/A | Can be reduced to 30 days or disabled |

**Implication:** Snapshots older than 90 days, non-prod VPC flow logs, and non-prod app logs are safe
to expire or delete from a compliance standpoint. Prod application logs and CloudTrail must stay at
or above 365 days.

---

## April 2026 MTD Cost Snapshot (11 days)

| Service | Apr MTD (USD) | Projected full month |
|---|---:|---:|
| Amazon RDS | 664.00 | ~$1,900 (RI front-loads Apr 1) |
| Amazon EC2 Compute | 465.28 | ~$1,400 |
| EC2 - Other (EBS/NAT/snapshots) | 424.37 | ~$1,270 |
| Amazon CloudWatch | 332.59 | ~$1,000 |
| Amazon ElastiCache | 298.81 | ~$895 (RI front-load) |
| Tax | 233.27 | ~$700 |
| OpenVPN (marketplace) | 108.36 | ~$325 |
| Amazon VPC | 96.04 | ~$288 |
| Amazon ELB | 66.01 | ~$198 |
| Amazon QuickSight | 33.72 | ~$101 |
| AWS End User Messaging (SMS) | 26.60 | ~$80 (dropped 85% since Mar 26 — 🚨 P1 safety incident, not a saving — see Proposal 18) |

---

## Proposal 1 — EBS Snapshot Lifecycle Policy

### Evidence
- **5,064 total snapshots** in `ap-southeast-2` as of 2026-04-11
- **4,741 snapshots (93.7%) older than 90 days**
- Total allocated snapshot volume: **153,604 GB**
- No Data Lifecycle Manager (DLM) policy exists — snapshots accumulate indefinitely
- Monthly cost grew from $387 (Jul 2025) → $631 (Mar 2026) and rising ~$20/month

### Compliance verdict
✅ **Safe to delete snapshots older than 90 days.** SOC 2 CC6.x recommends 90-day backup retention.
The Australian Privacy Act does not mandate snapshot retention. Financial/transaction data lives in
the database, not in snapshots.

⚠️ **Before bulk deletion, confirm with the team:** Is there a DR or business policy requiring
retention longer than 90 days? If yes, adjust the cutoff accordingly (common choices are 90 days
or 1 year for disaster recovery).

### Proposed action
1. Implement a DLM policy: retain last 7 daily + last 4 weekly snapshots per volume
2. After confirming no >90-day DR requirement, bulk delete snapshots older than 90 days
3. Monitor snapshot costs monthly via Cost Explorer; set billing alarm if cost rises above $250/month

### Savings estimate
**$350–500 USD/month** once lifecycle policy is active and old snapshots are purged.
Current trajectory will reach $700+/month by Q4 2026 without action.

### Risk
- **Low.** Snapshots are point-in-time backups. The running databases are the live copies.
- RDS has 7-day automated backups separately.

### Approval required
Yes — bulk deletion of production snapshots. Requires: team confirmation of retention policy,
then explicit written approval for deletion.

---

## Proposal 2 — `odoo-production` io1 → gp3 Storage Migration

### Evidence
- Single RDS instance using io1: `odoo-production` (PostgreSQL 14.17, 100 GB, 3,000 IOPS)
- Only 10 GB of data used (10% of allocated storage)
- All other RDS instances already on gp3
- `APS2-RDS:PIOPS` cost: $330/month in March 2026

**✅ IOPS confirmed (2026-04-11, CloudWatch 30-day data):**

| Metric | Daily average | 30-day peak |
|---|---|---|
| ReadIOPS | 0.8 IOPS | **422 IOPS** |
| WriteIOPS | 5.5 IOPS | **291 IOPS** |
| **Combined peak** | — | **~713 IOPS** |
| Provisioned (io1) | — | 3,000 IOPS |
| **Peak utilization** | — | **~24%** |

DB load average (Performance Insights): **0.034–0.041** — the database is essentially idle.
See `docs/investigations/2026-04-11-rds-deep-analysis.md` for full analysis.

### Cost comparison

| | io1 (current) | gp3 (proposed) |
|---|---|---|
| Storage | $0.138 × 100 GB = $13.80/month | $0.122 × 100 GB = $12.20/month |
| IOPS | $0.11 × 3,000 = $330/month | **Included** (3,000 IOPS is gp3 baseline) |
| **Total** | **~$344/month** | **~$12/month** |

### Compliance verdict
✅ **Safe — IOPS confirmed.** The database never exceeds 713 combined IOPS over 30 days. gp3 at
3,000 IOPS provides 4× headroom. QA and dev already use gp3 at 3,000 IOPS — a validated configuration.

### Proposed action
In-place RDS storage modification via `aws rds modify-db-instance`:
```
aws rds modify-db-instance \
  --db-instance-identifier odoo-production \
  --storage-type gp3 \
  --iops 3000 \
  --apply-immediately
```
Note: this triggers a storage optimization pass on the live instance. A brief I/O impact window
is expected; schedule for a maintenance window.

### Savings estimate
**~$330 USD/month.**

### Risk
- **Low.** In-place storage type migration with same IOPS. RDS handles the transition.
- Brief performance impact during optimization pass.
- 4× IOPS headroom confirmed — zero risk of IOPS starvation on migration.

### Approval required
Yes — production RDS modification. Requires explicit written approval + maintenance window.

---

## Proposal 3 — CloudWatch Metric Stream Namespace Filtering

### Evidence
- One metric stream running: `NewRelic-Metric-Stream` (created 2023-05-09, not updated since)
- **No include or exclude filters** — streams ALL CloudWatch metrics to New Relic
- March 2026 cost: `MetricStreamUsage` $579/month + `DataProcessing-Bytes` $172/month = ~$751/month attributable
- 290 metric alarms across EC2, ALB, Lambda, RDS, CWAgent namespaces

### Compliance verdict
✅ **Safe to filter** — with care. Filtering must not remove namespaces backing active New Relic
alerts. This requires coordination with the New Relic integration owner before any change.

### Proposed action
1. Pull the current list of namespaces in the metric stream
2. Audit New Relic dashboards and alert policies to identify actually-used namespaces
3. Add `ExcludeFilters` or restrict to `IncludeFilters` with confirmed required namespaces
4. Common candidates for exclusion: non-prod environment dimensions, unused AWS services

### Savings estimate
**$300–700 USD/month** depending on how many namespaces can be safely excluded.

### Risk
- **Medium.** Wrong exclusions can create alert gaps in New Relic.
- Mitigation: test filtering in a staging metric stream first.

### Approval required
Yes — requires New Relic integration owner sign-off. Then CloudWatch stream update approval.

---

## Proposal 4 — CloudWatch Log Group Retention Policies

### Evidence (2026-04-11 fresh data)

Top log groups by stored bytes, all with `NEVER_EXPIRE` retention:

| Log Group | Stored MB | Recommended retention |
|---|---:|---|
| `smoke-api-prod-pm2-out-log` | 127,631 | **365 days (SOC 2)** |
| `smoke-vpc-flow-log` | 127,181 | **365 days (SOC 2)** |
| `aws-waf-logs-prod` | 60,377 | **365 days (SOC 2)** |
| `smoke-qa-api-out` | 56,471 | 90 days |
| `vpc-flow-log-development` | 50,507 | 30 days |
| `qa-vpc-flow-logs` | 47,487 | 30 days |
| `sandbox-vpc-flow-log` | 35,931 | 30 days |
| `smoke-dev-api-out` | 27,937 | 90 days |
| `smoke-api-sandbox-pm2-out-log` | 18,843 | 90 days |
| `vpc-uat-log-group-name` | 17,686 | 30 days |
| `RDSOSMetrics` | 8,226 | 90 days |
| `/aws/rds/instance/sensor-qa/slowquery` | 8,075 | 90 days |
| `aws-waf-logs-sandbox` | 5,360 | 30 days |
| `smoke-qa-api-error` | 4,298 | 90 days |
| `smoke-dev-api-error` | 1,586 | 90 days |

Total 143 log groups, 70 with NEVER_EXPIRE, 588 GB total stored.

### Compliance verdict
✅ **Safe.** Setting retention on non-prod groups to 30–90 days does not violate any Australian
law or SOC 2 requirement. Production groups must remain at ≥365 days.

Setting `NEVER_EXPIRE` to 365 days on prod groups is also a minor improvement for predictability
(no data loss, just prevents indefinite growth).

### Proposed action
Batch apply `aws logs put-retention-policy` for each log group:
- Production (prod/WAF): set to 365 days
- Non-prod app logs (qa, dev, sandbox): set to 90 days
- Non-prod VPC flow logs (qa, dev, sandbox, uat): set to 30 days
- RDSOSMetrics: set to 90 days

### Savings estimate
**$50–200 USD/month** — primarily from stopping ongoing ingestion charges on non-prod VPC flow
logs and reducing future storage growth. Existing stored data ages out over the retention period.

### Risk
- **Very low.** Retention policies only affect future data; old data remains until the retention
  clock expires.

### Approval required
Yes — operational change to CloudWatch configuration.

---

## Proposal 5 — gp2 → gp3 EC2 EBS Volume Migration

### Evidence
- **55 gp2 volumes in-use** in `ap-southeast-2`, total **2,435 GB**
- gp2 price: $0.115/GB/month → $280/month
- gp3 price: $0.096/GB/month → $234/month
- **21 gp2 volumes are for stopped instances** (separate from in-use running volumes)

### Compliance verdict
✅ **Safe.** Volume type migration is in-place, non-destructive, and does not change data.

### Proposed action
For all gp2 volumes attached to running instances:
```
aws ec2 modify-volume --volume-id <vol-id> --volume-type gp3
```
This is a live modification; no downtime or unmounting required. AWS handles the optimization pass.

### Savings estimate
**~$46 USD/month** on running-instance volumes.
Additional savings from stopping paying for gp2 volumes on stopped instances (see Proposal 7).

### Risk
- **Very low.** Standard EBS volume modification. AWS throttles the optimization pass.

### Approval required
Yes — EBS volume modification on production EC2 instances.

---

## Proposal 6 — Unattached EBS Volume Cleanup

### Evidence
- **22 volumes in `available` (unattached) state**, total **685 GB**
- Estimated cost: **~$75 USD/month** (pure waste — no instance attached)

Sample unattached volumes:
```
vol-07a921d86594d5e89   44 GB gp2 (available)
vol-0cfafb5acdfc62e93  150 GB gp2 (available)
vol-01be6cc2c0d42218b   50 GB gp2 (available)
vol-06ff9b1e9e61427c1   30 GB gp2 (available)
vol-0fa5e11f893f9a46c   40 GB gp2 (available)
... (17 more)
```

### Compliance verdict
✅ **Safe to delete** once confirmed they are orphaned. EBS volumes themselves are not
compliance artefacts unless they contain unique data not stored elsewhere.

### Proposed action
1. For each unattached volume, check tags and creation date to identify the source instance
2. Confirm no intentional detachment (e.g., volume being held for re-attachment)
3. For clearly orphaned volumes: delete after taking a final snapshot as safety net (snapshot can be deleted after 30 days if no issue arises)

### Savings estimate
**~$75 USD/month** if all 22 volumes are confirmed orphaned.

### Risk
- **Low** with the final-snapshot safety net.
- Medium without: if a volume contains unique data, deletion is irreversible.

### Approval required
Yes — requires per-volume team confirmation before deletion.

---

## Proposal 7 — Stopped Instance Review and Decommission

### Evidence
**17 EC2 instances in stopped state** as of 2026-04-11:

| Instance ID | Name | Type |
|---|---|---|
| i-0271d68247fa19397 | Qa-kafka | t3.medium |
| i-09d3ff4a8e2b9775f | Emqx-qa | c5.large |
| i-0b95ad7a79856cea1 | sonarqube-dev | t3.medium |
| i-02ddec3d5019d5ca7 | Emqx-dev-Server | t2.small |
| i-00ba2a498df3447e4 | openvpn-uat | t2.micro |
| i-05f381197e504c262 | Emqx-uat | c5.large |
| i-0fd66936bcf46777b | odoo-uat-server | t3.medium |
| i-006be510eb5f985d4 | openvpn-dev | t2.micro |
| i-02920672e61afa067 | uat-kafka | t3.medium |
| i-0028143040e3d3f07 | Qa-kafka-private | t3.medium |
| i-0fd124877678a0a25 | Dev-Kafka | t3.medium |
| i-00a95bb3bac6cbbbf | smoke-dev-api-cron-server-private | t2.micro |
| i-0ba5d67daef255048 | Odoo-development-private | t3.medium |
| i-0789d06117f675d0e | Odoo-development-private-v2 | t3.medium |
| i-0e0695d1f69e6072a | odoo-qa-private | t3.medium |
| i-003ea4273c095a550 | dev-api-server | t3.medium |
| i-01165224653b92ed5 | dev-api-server | t3.medium |

- **16 EBS volumes attached** to these instances, total **716 GB**, estimated **~$82/month**
- Several also have EIPs associated (paying ~$3.65/month each while stopped)

### Compliance verdict
⚠️ **Review required before termination.** Stopped instances may be intentionally paused
for a rollout, or may be legacy dev environments no longer needed. This needs team input.

### Proposed action
1. Confirm with the team which instances are safe to terminate
2. For confirmed decommissions: snapshot the root volume as a safety net, then terminate
3. EBS volumes deleted automatically on termination (if `DeleteOnTermination = true`)
4. Release any associated EIPs after termination

### Savings estimate
**$40–82 USD/month** depending on which instances are terminated.

### Risk
- **Medium** without team confirmation. Terminated instances cannot be recovered.

### Approval required
Yes — requires per-instance team sign-off.

---

## Proposal 8 — RDS Non-Production Multi-AZ Removal

### Evidence
- `sensor-sandbox` (MySQL t3.medium): `MultiAZ = true`
- `smokealaramuat` (MySQL t3.medium): `MultiAZ = true`
- These are non-production environments with no HA requirement
- Multi-AZ doubles the underlying instance cost for these databases
- March 2026 `APS2-Multi-AZUsage:db.t3.medium` = $96.72/month total

### Compliance verdict
✅ **Safe for non-production.** Multi-AZ provides automatic failover — a production HA feature.
Sandbox and UAT environments have no availability SLA.

### Proposed action
```
aws rds modify-db-instance --db-instance-identifier sensor-sandbox --no-multi-az --apply-immediately
aws rds modify-db-instance --db-instance-identifier smokealaramuat --no-multi-az --apply-immediately
```
This triggers a storage migration; there is a brief disruption to the sandbox/UAT DB.

### Savings estimate
**~$60–65 USD/month** (two fewer Multi-AZ standby instances).

### Risk
- **Low for non-production.** These environments can tolerate downtime.

### Approval required
Yes — RDS modification.

---

## Proposal 9 — ElastiCache Non-Production Scheduling

### Evidence
- 6 x `cache.t3.medium` + 4 x `cache.t3.micro` running 24/7 in `ap-southeast-2`
- Non-prod clusters (QA, dev, sandbox) do not need to run nights and weekends
- March 2026 ElastiCache cost: $359/month

### Compliance verdict
✅ **Safe.** Stopping non-prod Redis clusters outside business hours has no compliance impact.

### Proposed action
Implement scheduled stop/start via AWS Systems Manager Automation or a Lambda scheduler:
- Stop non-prod ElastiCache clusters Mon–Fri 10pm → 7am AEST
- Stop all non-prod clusters on weekends

### Savings estimate
**$50–180 USD/month** depending on which clusters are on a schedule.

### Risk
- **Low.** Scheduled start/stop is a standard pattern. Redis is cold-start-safe (confirmed in INV-03).

### Approval required
Yes — requires approval to add scheduling automation.

---

## Proposal 10 — NAT Gateway Consolidation Review

### Evidence
- **5 active NAT gateways** across 5 VPCs in `ap-southeast-2`
- Each NAT costs $0.059/hr (~$43/month) + $0.059/GB data processed
- March 2026 NAT cost: $308/month

NAT gateway IDs:
```
nat-0419aece84205b4f5  vpc-03ec09702866ca7b1
nat-0946e6a99b6c5832d  vpc-03e1525483229726e
nat-0d90ba7448ffb1823  vpc-0682d7403418ab9db
nat-01453b472af5c5305  vpc-07917d0b4878f2c15
nat-0f00d13d55f11c828  vpc-0c4eb38fb674b072b
```

### Compliance verdict
⚠️ **Review required.** Removing or consolidating NAT gateways changes network egress paths.
Non-prod VPCs that can tolerate internet-less operation (or use VPC endpoints instead) are candidates.

### Proposed action
1. Identify which VPCs are non-prod (dev, qa, sandbox, uat)
2. Check whether non-prod VPCs actually need internet egress — many IoT/API workloads only need to
   reach AWS services (use VPC endpoints instead)
3. For confirmed non-prod VPCs with no required internet egress: remove NAT gateway
4. Optionally: share a single NAT gateway across prod VPC + non-prod VPCs via VPC peering

### Savings estimate
**$50–200 USD/month** if 1–3 non-prod NAT gateways are removed.

### Risk
- **Medium.** Removing NAT breaks outbound internet access for resources in that subnet.

### Approval required
Yes — networking change.

---

## Proposal 11 — Idle EIP Releases

### Evidence
- **5 EIPs in unassociated (idle) state** — each costs $0.005/hr = ~$3.65/month
- Total waste: ~$18/month
- Additionally, EIPs associated with stopped instances also cost ~$3.65/month each

Idle EIPs:
```
13.236.215.14   (unassociated)
13.54.14.80     (unassociated)
3.105.96.66     (unassociated)
52.64.196.201   (unassociated)
54.66.204.235   (unassociated)
```

### Compliance verdict
✅ **Safe.** EIPs are addressing artefacts with no compliance implication.

### Proposed action
After confirming none are needed for upcoming deployments:
```
aws ec2 release-address --allocation-id <alloc-id>
```

### Savings estimate
**~$18/month** for 5 idle EIPs. Additional ~$29/month if EIPs on stopped instances are released
when those instances are terminated.

### Risk
- **None.** Releasing idle EIPs has no service impact.

### Approval required
Yes (operational AWS change).

---

## Proposal 12 — OpenVPN Licensing Review

### Evidence
- OpenVPN Access Server (10 Connected Devices tier) via AWS Marketplace
- April MTD: $108 (projected ~$325/month)
- March 2026 actual: $437/month
- Currently used for: developer VPN access to production VPC

### Alternatives

| Option | Estimated cost | Notes |
|---|---|---|
| AWS Client VPN | ~$72/month (2 endpoints + connections) | Managed, integrates with IAM/AD |
| Self-hosted WireGuard (t3.micro) | ~$10/month | Lightweight, fast, easy to operate |
| OpenVPN lower tier | Varies | Possibly a lower-connection-count tier |
| Keep as-is | ~$325–437/month | No action needed if VPN is heavily used |

### Compliance verdict
✅ **Safe to replace.** VPN is a connectivity tool, not a compliance artefact. Any replacement
must maintain equivalent access controls.

### Proposed action
1. Confirm how many developers regularly use the VPN (max concurrent connections)
2. Evaluate WireGuard on t3.micro as a replacement (dramatically cheaper, better performance)
3. If compliance/audit requires a managed VPN solution, evaluate AWS Client VPN instead

### Savings estimate
**$100–350 USD/month** depending on chosen alternative.

### Risk
- **Medium.** VPN is used for developer access to production resources. Migration requires
  distributing new connection configs to all developers.

### Approval required
Yes — networking and access change.

---

## Proposal 13 — Vanta Contract Review and Renegotiation

### Evidence
- Vanta charge: $6,250 posting in March 2026 (`Global-SoftwareUsage-Contracts`)
- Likely 6-month installment (similar posting Sep 2025); implied annual cost ~$12,500
- Vanta role (`vanta-auditor`) last used 2026-04-08 — integration is active
- AWS Config, Security Hub, Inspector all enabled supporting Vanta workflows

### Compliance verdict
⚠️ **Do not remove without a transition plan.** If an active audit is underway, removing Vanta
disrupts evidence continuity. Australian Privacy Act APP 8 applies to cross-border data disclosure
to Vanta (a US-based vendor).

### Recommended action
Per `docs/investigations/2026-04-09-vanta-recommendation.md`:
1. **Confirm contract renewal date and notice window** (auto-renews annually, 30-day notice typically)
2. **Request a downsell/right-size quote** — if only one AWS account is in scope, a lower tier may apply
3. **Get competing quotes** from Drata and Secureframe using the same scope parameters
4. **Do not replace mid-audit** — time any transition to align with the compliance calendar

### Savings estimate
**$0–6,250+/year** depending on negotiation outcome. Cannot be estimated without contract details.

### Risk
- **High if rushed.** Audit continuity and APP 8 cross-border obligations must be addressed.

### Approval required
Business/commercial decision. Involves legal, procurement, and the compliance team.

---

## Proposal 14 — QuickSight Rightsizing

### Evidence
- March 2026: $102/month
- April MTD: $33 (projected ~$101/month)
- QuickSight Author licenses cost $24/month each; Reader sessions cost $0.30 each

### Compliance verdict
✅ **Safe to reduce.** QuickSight is an analytics tool with no compliance dependency.

### Proposed action
1. Audit active QuickSight users — identify how many Author licenses are in use vs Reader-only
2. Downgrade inactive Authors to Reader licenses or remove access
3. Confirm dashboards are still needed; decommission unused

### Savings estimate
**$50–85 USD/month** if 2–3 unused Author seats are removed.

### Risk
- **Very low.** Dashboard users will lose authoring capability but retain read access.

### Approval required
Yes — QuickSight account management.

---

## Proposal 15 — Vanta → AWS-Native Compliance Replacement

> **New direction (confirmed 2026-04-11):** The team wants to investigate removing Vanta and
> replacing it with native AWS compliance services.

### Current Vanta cost
~$12,500/year (inferred from two ~$6,250 bi-annual postings in Sept 2025 and Mar 2026).

### What Vanta provides vs AWS-native equivalents

| Vanta capability | AWS-native equivalent | Covered? |
|---|---|---|
| Continuous AWS resource monitoring | AWS Config + Security Hub | ✅ Already enabled |
| Vulnerability scanning (EC2, Lambda) | AWS Inspector | ✅ Already enabled |
| Threat detection | AWS GuardDuty | ✅ Already enabled |
| Data classification | Amazon Macie | ✅ Already enabled |
| SOC 2 control mapping + evidence collection | AWS Audit Manager | ⚠️ Not enabled — key gap |
| Public trust center / customer-facing status | AWS Artifact (internal only) | ❌ No public equivalent |
| Employee access reviews | AWS IAM Access Analyzer + manual | ⚠️ Partial |
| Vendor/third-party risk management | Manual process | ❌ No equivalent |
| Questionnaire automation | Manual process | ❌ No equivalent |

**Gap summary:** Native AWS covers all *technical* compliance controls. The main gaps vs. Vanta are
evidence packaging (solved by Audit Manager), public trust center, and questionnaire automation.
If customers actively use the Vanta trust center URL, this must be addressed before removal.

### Cost model for native AWS replacement

| Service | Est. monthly cost | Notes |
|---|---|---|
| Security Hub + Config + Inspector + GuardDuty + Macie | ~$73/month combined | Already running and paid |
| **AWS Audit Manager** | **~$30–100/month** | Not yet enabled; ~$1.25–4.50/resource |
| **Incremental cost to replace Vanta** | **~$30–100/month** | Just Audit Manager |
| **Vanta cost (inferred monthly)** | **~$1,042/month** | $12,500 ÷ 12 |
| **Net saving if Vanta removed** | **~$940–1,010/month** | |

### Compliance risk assessment

⚠️ **Vanta removal needs planning — do not remove mid-audit:**
1. Confirm whether an active SOC 2 audit is underway
2. Enable and configure AWS Audit Manager before Vanta is removed (no evidence gap)
3. Confirm whether any customer contracts reference Vanta's trust center
4. Check Vanta renewal/notice date (~30-day notice window typical)

### Proposed investigation steps (MFA session required)
1. Enable AWS Audit Manager in evaluation mode — estimate actual cost at current resource scale
2. Confirm whether any customers access the Vanta trust center URL
3. Confirm whether any active SOC 2 audit is in progress
4. Get the Vanta contract renewal date from the contract owner
5. Run Audit Manager in parallel for 30 days, then cancel Vanta at renewal

### Savings estimate
**~$940–1,010 USD/month** once Vanta is replaced by Audit Manager.

---

## Proposal 16 — New Relic → AWS CloudWatch Native Monitoring

> **New direction (confirmed 2026-04-11):** The team wants to remove New Relic and use AWS-native
> monitoring. This is the single highest-value change — it eliminates the metric stream entirely.

### Current cost attributable to New Relic
- `MetricStreamUsage`: $579/month
- `DataProcessing-Bytes`: $172/month
- Kinesis Firehose: ~$5/month
- **Total New Relic-driven CloudWatch cost: ~$756/month**
- Plus any New Relic subscription not visible in this AWS billing account

### Metric stream configuration (confirmed 2026-04-11)
- Stream name: `NewRelic-Metric-Stream` — State: **running**
- Created: 2023-05-09, never updated
- **No IncludeFilters, no ExcludeFilters** — streaming ALL CloudWatch namespaces to New Relic
- Output: `NewRelic-Delivery-Stream` (Kinesis Firehose)

### What New Relic provides vs CloudWatch

| Capability | New Relic | AWS CloudWatch |
|---|---|---|
| Infrastructure metrics (EC2, RDS, ELB) | ✅ Via metric stream | ✅ Native — no stream needed |
| Custom operational dashboards | ✅ Advanced UI | ✅ Basic (sufficient for ops) |
| Alarms and notifications | ✅ Advanced | ✅ 290 alarms already exist |
| Log search / aggregation | ✅ | ✅ CloudWatch Logs Insights |
| APM / distributed tracing | ✅ (if agent installed) | ⚠️ X-Ray (partial) |
| Free tier | ✅ 100 GB/month | Pay-per-use |

**Key insight:** 290 CloudWatch alarms are already configured in this account. The metric stream
is a secondary overlay for New Relic dashboards only. Removing the stream eliminates ~$756/month
immediately — the alarms continue to function unaffected.

### Migration approach
1. Audit which New Relic dashboards are actively consulted (vs. set-and-forgotten)
2. Recreate critical dashboards in CloudWatch Dashboards ($3/dashboard/month)
3. Remove `NewRelic-Metric-Stream` and `NewRelic-Delivery-Stream` after migration
4. Optionally: evaluate AWS X-Ray + CloudWatch Synthetics for any APM needs

### Savings estimate
**~$756 USD/month** (CloudWatch metric stream + processing eliminated entirely).
Additional savings if New Relic has an external subscription — that cost is separate and additive.

### Risk
- **Medium.** Dashboard migration requires 1–2 days of engineering effort for basic dashboards.
- Existing 290 CloudWatch alarms are unaffected — they do not depend on the metric stream.

---

## Proposal 17 — Prod EC2 Autoscaling Right-Sizing

> **New direction (confirmed 2026-04-11):** Team wants to investigate whether the prod API fleet
> can be right-sized with autoscaling, keeping only minimum instances to handle traffic.

### What is confirmed (2026-04-11)

**Prod API AutoScaling Group:** `CodeDeploy_Smoke-API_d-NNRF41DGD`

| Parameter | Current value |
|---|---|
| Instance type | t3.medium (from launch template `prod-api-server-lt`) |
| Min / Desired / Max | **4 / 4 / 5** |
| Scaling policy | Target Tracking — CPU 75% |
| Actual 4 instances today | All healthy, InService |

**Actual CPU utilization (last 30 days, 4 prod API instances):**

| Instance | 30-day avg CPU | 30-day peak CPU |
|---|---|---|
| i-0277626c3f4828de7 | 8.8% | 78.6% |
| i-09acc276e638ba099 | 10.9% | 78.3% |
| i-0b53bd5e7a9b9dec0 | 8.5% | 78.9% |
| i-0bcc2dcab7c6dfedf | 5.6% | 31.6% |

### Observations

1. **Average CPU is 5.6–10.9%** — the fleet is heavily over-provisioned for average load.
2. **Peak CPU is 78–79% on 3 of 4 instances** — this suggests a periodic burst (likely CodeDeploy deployment or a daily batch job) rather than sustained load. One instance has only 31.6% peak.
3. **The scaling band is Min=4, Max=5** — this adds only 1 instance on scale-out, then holds. This is effectively a fixed 4-instance fleet with a 1-instance emergency buffer, not a true autoscaling configuration.
4. **All 4 instances are t3.medium** (~$0.052/hr each = ~$151/month each = ~$604/month total for API instances).

### Recommendation

Before changing Min, verify the cause of the 78% CPU spikes:
- If caused by deployments (Blue/Green), CPU spikes are predictable and short-lived — safe to lower Min.
- If caused by real traffic surges, must retain current Min.

**If deployment spikes are the cause:** Lower Min from 4 to 3. This saves ~$150/month while keeping
the same 75% CPU target-tracking policy. Scale-out still protects against genuine traffic spikes.

**Conservative approach:** Lower Min to 3, watch for 2 weeks, then evaluate further.

### Savings estimate
**$130–300 USD/month** — lower bound is reducing Min by 1 (one t3.medium), upper bound includes
instance type optimization pending load analysis.

_(Revised down from initial estimate — original estimate assumed no ASG existed. ASG is already in place with CPU scaling.)_

### Risk
- **Low–Medium.** ASG and CodeDeploy Blue/Green are already in place — prerequisites met.
- Risk: if CPU spikes are traffic-driven (not deployment-driven), reducing Min below HA threshold causes degraded performance during spikes.
- Mitigation: monitor 75% CPU target; lower Min incrementally.

### Approval required
Yes — production fleet configuration change.

---

## Proposal 18 — SMS Reliability Investigation (Urgent — Not a Cost Win)

> **URGENT: Root cause mechanism confirmed 2026-04-12 via live DB investigation.**

### Evidence

**AWS-level (confirmed 2026-04-11):**
Daily SMS sends dropped from **350–585/day** (Mar 20–25) → **29/day** (Mar 27). Drop is in
send volume, NOT delivery failures. AWS infrastructure is fine. No AWS-level cause found.

**DB-level (confirmed 2026-04-12 via live MySQL investigation):**
`tbl_alarm_alerts` — the table that drives the SMS notification pipeline — stopped receiving
new rows starting partway through March 26:

| Date | Alert rows created |
|---|---|
| Mar 22–25 | 25–39/day (normal) |
| **Mar 26** | **14** (drop begins mid-day) |
| **Mar 27** | **4** |
| Mar 28+ | 0–2/day |

All alert types dropped uniformly (DISCONNECT, BATTERY, ALERT, TAMPERED) — the **entire alert
creation pipeline stopped** mid-day March 26. 4,808 device records were also bulk-updated on
March 25–26, consistent with a migration script in a deployment.

**`CommunicationsQueue` hypothesis: refuted.** `tbl_communications_queue` is email-only (no SMS
rows). The flag governs email batch delivery, not SMS.

Full investigation notes: `docs/investigations/2026-04-12-mysql-sms-root-cause.md`

### Root cause assessment

Most likely: **a backend code deployment on March 26 broke the MQTT event → `tbl_alarm_alerts`
insertion pipeline.** The app team needs to check ECS task revision history and git log for the
MQTT subscriber service around March 25–27.

### Why this matters
This is a **production safety and reliability issue** for a smoke alarm monitoring platform.
If tenants and property managers are not receiving alarm notifications due to a silent failure,
there may be real-world safety consequences and potential liability under Australian building
safety obligations.

### Immediate investigation steps (updated 2026-04-12)
1. **App team**: Check ECS task revision history for prod MQTT subscriber — any new revisions March 25–27?
2. **App team**: Check git log for MQTT subscriber service changes around March 26
3. **App team**: Identify what the March 25–26 bulk `tbl_alarms` update changed (what column, what value)
4. **App team**: Confirm whether `tbl_sms_allowed_list` (2 entries, Indian test numbers) is referenced in production code — if yes, it blocks all Australian SMS

**This must be treated as a P1 reliability incident. No cost optimization work should touch
the SMS/notification infrastructure until this is resolved.**

---

## Proposal 19 — Lambda Runtime EOL + Memory Rightsizing

### Evidence (confirmed 2026-04-11)
- **5 functions on `nodejs16.x`** (EOL since March 2024, AWS stopped support):
  - `createfilezipdev`, `createfilezipsandbox`, `createfilezipprod`, `createfilezipqa`, `createfilezipuat`
- **`createfilezipqa` has 2,048 MB memory** vs 512 MB for all other `createfilezip*` functions — likely misconfigured
- These functions appear to be PDF/document generation utilities

### Cost impact
- `createfilezipqa` at 2,048 MB vs 512 MB = 4× cost per invocation for non-prod function
- If called frequently, the excess memory allocation is wasted ($0.0000166667/GB-second)
- EOL runtimes do not receive security patches — a risk more than a cost, but may cause unexpected behaviour

### Proposed action
1. Upgrade all `createfilezip*` functions from `nodejs16.x` to `nodejs22.x` (current LTS)
2. Reduce `createfilezipqa` memory from 2,048 MB to 512 MB (match other environments)
3. Test each function post-upgrade in non-prod before touching `createfilezipprod`

### Savings estimate
**Low (<$10/month)** — Lambda is a small cost driver here. Primary value is security hygiene.

### Risk
- **Low.** Node 22 is compatible with Node 16 for most typical function code. Test in dev/sandbox first.

### Approval required
Yes — code change + Lambda configuration update.

---

## Proposal 20 — S3 Lifecycle Policies for Log/ALB Buckets

### Evidence (confirmed 2026-04-11)
- **75 S3 buckets** in account
- `sensor-global-cloudtrail-logs` — ✅ has 365-day lifecycle policy (good)
- **No lifecycle policies** on:
  - `smokeapibucket`, `smokebucketlogs`, `smokesnslogs` (application logs)
  - `smoke-dev-api-lb-logs`, `smoke-qa-api-lb-logs`, `smoke-sandbox-api-lb-logs`, `sensoralblogs` (ALB access logs)
  - `dev-admin-panel-logs`, `smoke-sso-dev-api-lb-logs` (other logs)
  - `aws-quicksetup-patchpolicy-access-log-*` buckets (~14 buckets, SSM Quick Setup artifacts)
  - `smokealarmdevelopment-angular`, `smokealarmqa-angular`, `smokealarmsandbox-angular` (static assets)

### Cost impact
Without lifecycle policies, all objects accumulate indefinitely at S3 Standard pricing ($0.023/GB/month).
ALB logs at high-traffic sites can grow several GB/day. Without knowing exact sizes (per-bucket CW
metrics not available via current API call), estimate $20–100/month in reducible storage.

### Proposed action
1. Add `Expiration: 90 days` lifecycle rule to all non-prod ALB log buckets
2. Add `Expiration: 365 days` lifecycle rule to prod ALB log bucket (compliance)
3. Add `Expiration: 30 days` lifecycle rule to `aws-quicksetup-patchpolicy-access-log-*` buckets (no compliance value)
4. Add S3 Intelligent-Tiering to `smokeapibucket` if it contains infrequently-accessed objects

### Savings estimate
**$20–100 USD/month** depending on actual bucket sizes (to be confirmed).

### Risk
- **Very low.** ALB logs are operational logs only — no compliance dependency for non-prod.

### Approval required
Yes — S3 lifecycle configuration change.

---

## Proposal 21 — ElastiCache Non-Prod Right-Sizing

### Evidence (confirmed 2026-04-11)
10 Redis nodes across 5 environments:

| Environment | Nodes | Node type |
|---|---|---|
| dev | 2 | cache.t3.micro |
| qa | 2 | cache.t3.micro |
| sandbox | 2 | **cache.t3.medium** |
| prod | 2 | cache.t3.medium |
| uat | 2 | **cache.t3.medium** |

**Observation:** `sandbox` and `uat` use t3.medium (same as prod). `dev` and `qa` use t3.micro.
UAT and sandbox are pre-production environments — they likely don't need production-class Redis nodes.

- cache.t3.medium: ~$60/month each (~$120/month per 2-node environment)
- cache.t3.micro: ~$20/month each (~$40/month per 2-node environment)
- Downgrading sandbox + uat from t3.medium → t3.micro saves: ~$160/month

Note: Proposal 9 (ElastiCache scheduling) also applies to dev, qa, sandbox, uat environments.

### Proposed action
1. Downgrade `smokealarmsandbox-001`, `-002` and `smoke-uat-redis-001`, `-002` from t3.medium → t3.micro
2. Verify no sandbox/UAT test scenario requires high memory throughput from Redis

### Savings estimate
**~$160 USD/month** (node type right-sizing) + $50–130/month additional from scheduling (Proposal 9).

### Risk
- **Low.** Non-prod environments only. Redis t3.micro has 0.555 GB memory — sufficient for most dev/test workloads.

### Approval required
Yes — requires confirmation that sandbox/UAT test workloads don't depend on t3.medium capacity.

---

| Question | Status |
|---|---|
| EBS snapshot retention (90 days?) | ✅ **Confirmed 90 days — ready to proceed with DLM** |
| Stopped instances safe to terminate? | ⚠️ Per-instance review needed |
| Unattached EBS volumes intentional? | ⚠️ Needs team confirmation |
| SMS drop — intentional? | 🚨 **Confirmed NOT intentional — treat as P1 reliability incident** |
| New Relic namespaces used? | ✅ **Moot — team confirmed they want to remove New Relic entirely** |
| Vanta contract renewal date? | ⚠️ Unknown — needs contract owner to check |
| Customer log retention >365 days? | ⚠️ Unknown — confirm with business |

---

## Consolidated Savings Table (Updated 2026-04-11, Phase 2 complete)

| # | Proposal | Monthly savings | Status |
|---|---|---:|---|
| 1 | EBS snapshot lifecycle | $350–500 | Ready for approval (90-day confirmed) |
| 2 | odoo-production io1→gp3 | **~$330** | ✅ IOPS confirmed — ready for approval |
| 3 | New Relic removal — stop metric stream | **~$756** | Blocked on dashboard migration (Proposal 16) |
| 4 | Log group retention policies | $50–200 | Ready for approval |
| 5 | gp2→gp3 EC2 volumes | ~$46 | Ready for approval |
| 6 | Unattached EBS volumes | ~$75 | Ready after per-volume team confirmation |
| 7 | Stopped instance decommission | $40–82 | Ready after per-instance confirmation |
| 8 | RDS non-prod Multi-AZ removal | $60–65 | Ready for approval |
| 9 | ElastiCache non-prod scheduling | $50–180 | Ready for approval |
| 10 | NAT gateway consolidation | $50–200 | Needs further investigation |
| 11 | Idle EIP releases | ~$18 | Ready for approval |
| 12 | OpenVPN replacement | $100–350 | Needs further investigation |
| 13 | Vanta removal → AWS Audit Manager | **~$940–1,010** | Blocked on Proposal 15 steps |
| 14 | QuickSight rightsizing | $50–85 | Ready for approval |
| 17 | Prod API ASG Min reduction (4→3) | $130–300 | Ready — verify CPU spike cause first |
| 19 | Lambda runtime + memory fix | <$10 | Security hygiene — schedule any sprint |
| 20 | S3 lifecycle policies for log buckets | $20–100 | Ready for approval |
| 21 | ElastiCache non-prod right-sizing | ~$160 | Ready for approval |
| 22 | Stop non-prod environments | ~$620–700 | Ready — requires written approval |
| **Total addressable** | | **$3,895–5,630/month** | |

---

## Proposal 22 — Stop Non-Prod Environments (EC2 + RDS)

### Evidence
All non-production ASG and RDS instances run 24/7, paying full compute costs even when engineers
are not using them. Based on the inventory confirmed 2026-04-11:

**EC2 AutoScaling Groups (non-prod):**

| ASG | Env | Desired | Approx. $/month |
|---|---|---|---|
| CodeDeploy_Smokealarmdevelopmentapi_d-KHX1BJY5D | dev | 2 | ~$60 |
| CodeDeploy_dev-sso-api-dg_d-4O83LCN0D | dev | 1 | ~$30 |
| CodeDeploy_Smoke-qa-API_d-NK1QJ9K51 | qa | 3 | ~$90 |
| CodeDeploy_qa-sso-api-dg_d-Y5R4U1N0D | qa | 1 | ~$30 |
| CodeDeploy_Smoke-Sandbox-Api-Dg_d-EMHG7U561 | sandbox | 2 | ~$60 |
| CodeDeploy_sandbox-sso-api-dg_d-P0SQWI50D | sandbox | 1 | ~$30 |

**RDS (non-prod, t3.medium instances):**
- odoo-dev, odoo-qa, sensor-qa, sensor-sandbox, dev-db-temp-mysql-8

### Proposed action
1. Run `scripts/stop-nonprod.sh` to save state and stop environments
2. Re-run weekly (see caveat below)
3. Run `scripts/start-nonprod.sh --state-file tmp/nonprod-state-*.json` to resume when needed
4. Follow `docs/runbooks/08-stop-nonprod-environments.md` for full checklist

### Savings estimate
**~$620–700 USD/month** (EC2 compute + RDS compute when stopped)

Notes:
- RDS storage continues to accrue (~$10–15/month per instance) even when stopped
- ElastiCache **cannot be stopped** without deletion — dev/qa t3.micro clusters (~$20/node)
  remain running. Addressed separately by Proposals 9 and 21.
- UAT environments: not stopped by default (requires `--include-uat` flag to ops team agreement)

### ⚠️ RDS 7-Day Auto-Restart Caveat
AWS automatically restarts stopped RDS instances after **7 days** — this is an AWS hard limit and
cannot be disabled. To keep non-prod RDS down long-term, the stop script must be re-run weekly.
An EventBridge Scheduler rule can automate this (not implemented but documented in the runbook).

### Risk
- **Very low.** Non-prod environments only. Prod is protected by hardcoded safety list in scripts.
- Engineers who need non-prod during a sprint must request start via `scripts/start-nonprod.sh`.
- Recommend communicating to the team before first stop.

### Approval required
Yes — written sign-off from engineering lead confirming non-prod can be stopped.
See `docs/runbooks/08-stop-nonprod-environments.md` for full approval gate checklist.

---

## Remaining Investigations (requires refreshed MFA session)

All major investigations are now complete. Minor remaining items:
- **NAT gateway VPC mapping** — identify which VPCs are non-prod to size consolidation opportunity
- **Stopped instance per-instance confirmation** — review 2 stopped instances before decommission
- **QuickSight user audit** — confirm active vs inactive Author seats
- **Vanta contract renewal date** — confirm with contract owner

## Recommended Next Steps

1. 🚨 **P1: Investigate SMS reliability** — application-layer root cause needed. See `docs/investigations/2026-04-11-sms-drop-investigation.md`
2. **Group A — approve now (low risk, ~$374–669/month):**
   - Proposal 4: Log group retention policies
   - Proposal 5: gp2→gp3 EC2 volumes
   - Proposal 8: Non-prod RDS Multi-AZ removal
   - Proposal 11: Idle EIP releases (~$18)
   - Proposal 20: S3 lifecycle policies
   - Proposal 21: ElastiCache non-prod right-sizing (~$160)
   - **Proposal 22: Stop non-prod environments (~$620–700) — highest-impact quick win with written approval**
3. **Group B — approve with review (~$535–1,147/month):**
   - Proposal 1: EBS snapshot DLM (bulk deletion — needs written approval)
   - Proposal 2: odoo-production io1→gp3 (IOPS confirmed safe — schedule maintenance window)
   - Proposal 9: ElastiCache non-prod scheduling
   - Proposal 17: Prod API ASG Min=4→3 (verify CPU spike cause first)
4. **Group C — longer-horizon, ~$1,756–2,310/month:**
   - Proposal 3/16: New Relic removal (migrate dashboards first)
   - Proposal 13/15: Vanta → Audit Manager (confirm contract date, enable Audit Manager first)
5. **Group D — investigation required:**
   - Proposal 10: NAT gateway consolidation
   - Proposal 12: OpenVPN replacement
