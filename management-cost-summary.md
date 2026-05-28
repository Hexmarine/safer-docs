# AWS Cost Optimisation — Summary for Management Review

**Prepared:** 2026-04-11  
**Prepared by:** Engineering  
**Account:** AWS `747293622182` (ap-southeast-2, production IoT smoke alarm platform)  
**All figures:** AUD (1 USD ≈ 1.58 AUD, April 2026, rounded to nearest A$50/A$100)

> **Historical status (updated 2026-04-27):** This summary was prepared before UAT, QA, and Dev were fully decommissioned on 2026-04-21/22. Keep it as the original management view, but use `docs/investigations/2026-04-20-infra-inventory.md` for current prod+sandbox steady-state costs and completed savings.

---

## Current Spend vs Opportunity

| | Amount (AUD) |
|---|---|
| Estimated monthly AWS spend (March 2026) | ~A$12,000–13,500/mo |
| Total addressable saving identified | ~A$6,200–8,900/mo |
| Saving as % of current spend | ~47–66% |

Full technical detail: `docs/investigations/2026-04-11-cost-optimization-proposals.md`

---

> ### 🚨 P1 Reliability Alert — SMS Notifications (action required separately)
> A 93% drop in outgoing SMS notifications was confirmed on **March 27, 2026** (from ~380/day to ~29/day).
> AWS infrastructure is healthy — this is an **application-level issue** (feature flag, data change, or config).
> **No cost optimisation work should touch the SMS or notification infrastructure until resolved.**
> Investigation notes: `docs/investigations/2026-04-11-sms-drop-investigation.md`

---

## Cost Optimisation Proposals — Sorted by Saving

| # | Item | ~Monthly saving | Risk | Eng. effort | What changes | Status |
|---|---|---:|---|---|---|---|
| 13/15 | Remove Vanta, replace with AWS-native compliance tools | ~A$1,500 | Medium | High | Cancel Vanta SaaS contract (~A$1,500–1,600/mo); enable AWS Audit Manager + Config Rules | 🔒 Blocked — confirm contract renewal date first |
| 3/16 | Remove New Relic, switch to CloudWatch native monitoring | ~A$1,200 | Medium | High | Stop CloudWatch metric stream to New Relic; migrate alert dashboards to CloudWatch | 🔒 Blocked — migrate dashboards first |
| 22 | Stop non-production environments when not in use | ~A$1,000 | Low | Low | Scripts already written — zero dev/qa/sandbox EC2 ASGs and stop RDS after hours; resume on demand | ✅ Ready — written approval needed |
| 1 | Delete EBS snapshots older than 90 days; automate retention | ~A$700 | Low | Low | Create AWS Data Lifecycle Manager policy; one-time bulk delete of old snapshots | ✅ Ready — 90-day retention confirmed compliant |
| 2 | Migrate `odoo-production` database disk from io1 → gp3 | ~A$500 | Medium | Low | Change RDS storage type during a maintenance window; 30-min downtime | ✅ Ready — IOPS usage confirmed at 24% of provisioned |
| 12 | Replace OpenVPN EC2 instance with AWS Client VPN or SSM | ~A$350 | High | High | Decommission self-managed OpenVPN; migrate developer access to managed service | 🔍 Needs investigation |
| 17 | Reduce production API fleet minimum instance count (4 → 3) | ~A$350 | Medium | Low | Lower ASG minimum; saves one t3.medium instance; autoscaling still active | ✅ Ready — verify CPU spike root cause first |
| 21 | Right-size sandbox and UAT Redis cache nodes | ~A$250 | Low | Low | Downgrade cache.t3.medium → cache.t3.micro for sandbox and UAT (non-prod only) | ✅ Ready — dev/qa already on t3.micro with no issues |
| 4 | Set retention policies on CloudWatch log groups | ~A$200 | Low | Low | Apply 30-day retention to non-prod logs; 90-day to prod app logs; CloudTrail stays at 365 days | ✅ Ready — no compliance impact confirmed |
| 10 | Consolidate NAT gateways across VPCs | ~A$200 | Medium | Medium | Reduce number of NAT gateways by sharing across non-prod VPCs | 🔍 Needs investigation — VPC topology mapping needed |
| 9 | Schedule non-prod Redis cache clusters off overnight | ~A$200 | Low | Medium | Lambda + EventBridge rule to delete and recreate dev/qa ElastiCache clusters on a schedule | ✅ Ready — no IaC exists yet (engineering effort to build) |
| 6 | Remove unattached EBS volumes | ~A$100 | Low | Low | Delete unattached volumes confirmed as unused after team review | ✅ Ready — per-volume confirmation from team needed |
| 14 | Right-size QuickSight to fewer Author seats | ~A$100 | Low | Low | Reduce billable Author licences to active users only | ✅ Ready — user audit needed |
| 8 | Remove Multi-AZ from sandbox and UAT databases | ~A$100 | Low | Low | Disable standby replica on non-prod RDS; maintenance window required | ✅ Ready — non-prod only |
| 7 | Decommission stopped EC2 instances | ~A$100 | Low | Low | Terminate stopped instances confirmed as no longer needed | ✅ Ready — per-instance confirmation from team needed |
| 20 | Add expiry policies to S3 log buckets | ~A$100 | Low | Low | Apply 90-day lifecycle rule to ALB and application log buckets (non-prod); 365-day to prod | ✅ Ready — no compliance impact on non-prod |
| 5 | Migrate EC2 EBS volumes from gp2 → gp3 | ~A$50 | Low | Low | Online volume type migration — no downtime, faster and cheaper than gp2 | ✅ Ready — zero risk, online migration |
| 11 | Release 5 idle Elastic IP addresses | ~A$50 | Low | Low | Release EIPs confirmed as unused | ✅ Ready — per-IP confirmation from team needed |
| 19 | Upgrade Lambda functions from Node.js 16 (EOL) | negligible | Low | Low | Update runtime to Node.js 22; fix one memory misconfiguration | ✅ Ready — primarily a security hygiene item |
| | **Total addressable** | **~A$7,100/mo** | | | | |

> **Range:** ~A$6,200–8,900/mo depending on which items are approved and actual resource usage.

---

## What "Ready" Means

**✅ Ready** — technical investigation is complete, data supports the change, scripts or runbooks are prepared.  
Engineering can execute within days of written approval.

**🔒 Blocked** — depends on a prior step (e.g. migrating dashboards before removing the tool that drives them).  
These can run in parallel with the ready items while blockers are resolved.

**🔍 Needs investigation** — more technical analysis required before a safe change can be designed.

---

## Risk and Effort Key

| Level | Risk meaning | Effort meaning |
|---|---|---|
| **Low** | Non-prod only, or online/reversible change with no customer impact | Half a day or less to execute |
| **Medium** | Production resource affected, or requires a maintenance window | 1–3 days including testing |
| **High** | Major infrastructure change, risk of broader service disruption | Multi-week project |

---

## Recommended Sequence

1. **Approve Group A immediately** (~A$2,500–2,700/mo, all low-risk):
   - Stop non-prod environments (#22) — biggest single saving, scripts ready
   - Log retention (#4), S3 lifecycle (#20), gp2→gp3 EC2 volumes (#5)
   - Multi-AZ removal (#8), ElastiCache right-sizing (#21), idle EIPs (#11), QuickSight (#14)

2. **Schedule Group B for next maintenance window** (~A$1,200/mo):
   - Snapshot DLM + old snapshot cleanup (#1)
   - `odoo-production` io1→gp3 migration (#2) — IOPS confirmed safe

3. **Unblock and execute Group C** (~A$2,700–2,800/mo):
   - Vanta → AWS Audit Manager (#13/15) — confirm contract date, then proceed
   - New Relic removal (#3/16) — migrate dashboards, then stop metric stream

4. **Investigate and design Group D**:
   - NAT gateway consolidation (#10), OpenVPN replacement (#12)

---

*Full technical details, runbooks, and rollback plans: `docs/investigations/2026-04-11-cost-optimization-proposals.md`*  
*Change verification health check: `scripts/health-check.sh`*  
*Non-prod stop/start automation: `scripts/stop-nonprod.sh` / `scripts/start-nonprod.sh`*

---

## Quick Reference List

1. Remove Vanta, replace with AWS-native compliance (Audit Manager + Config)
   ~A$1,500/mo

2. Remove New Relic, migrate alerts to CloudWatch native monitoring
   ~A$1,200/mo

3. Stop non-production environments (dev/qa/sandbox) when not in use
   ~A$1,000/mo

4. Delete EBS snapshots older than 90 days; automate retention going forward
   ~A$700/mo

5. Migrate `odoo-production` database disk from io1 → gp3 (maintenance window)
   ~A$500/mo

6. Replace self-managed OpenVPN EC2 instance with AWS Client VPN or SSM
   ~A$350/mo

7. Reduce production API fleet minimum size from 4 → 3 instances (autoscaling stays on)
   ~A$350/mo

8. Right-size sandbox and UAT Redis cache nodes from t3.medium → t3.micro
   ~A$250/mo

9. Set retention policies on CloudWatch log groups (30-day non-prod, 90-day prod)
   ~A$200/mo

10. Consolidate NAT gateways across non-production VPCs
    ~A$200/mo

11. Schedule non-prod Redis cache clusters to shut off overnight
    ~A$200/mo

12. Remove unattached EBS volumes confirmed as unused
    ~A$100/mo

13. Right-size QuickSight to active Author seats only
    ~A$100/mo

14. Remove Multi-AZ standby from sandbox and UAT databases
    ~A$100/mo

15. Terminate stopped EC2 instances confirmed as no longer needed
    ~A$100/mo

16. Add expiry lifecycle policies to S3 log and ALB access-log buckets
    ~A$100/mo

17. Migrate EC2 EBS volumes from gp2 → gp3 (online, no downtime)
    ~A$50/mo

18. Release 5 Elastic IP addresses that are allocated but unused
    ~A$50/mo

19. Upgrade Lambda functions from Node.js 16 (end-of-life) to Node.js 22
    negligible (primarily a security item)

---
**Total: ~A$7,100/mo** · range A$6,200–8,900/mo  
*1 USD ≈ 1.58 AUD (April 2026), rounded to nearest A$50/A$100*
