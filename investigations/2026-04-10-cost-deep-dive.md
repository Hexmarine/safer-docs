# Cost Deep Dive — April 2026

- Date: 2026-04-10
- Scope: updated cost baseline for February–April 2026, with drill-downs on EBS snapshots, PIOPS, SMS anomaly, ElastiCache, and the April 1 billing spike
- Related systems: AWS Cost Explorer, EC2, EBS, RDS, ElastiCache, CloudWatch, AWS End User Messaging, OpenVPN

## Objective

Build on the March 2026 cost quick-wins baseline with fresh AWS Cost Explorer data, explain the April 1 spike, and identify new or updated savings opportunities.

## Sources Used

- AWS commands:
  - `aws ce get-cost-and-usage` — monthly and daily granularity for Feb–Apr 2026, grouped by SERVICE and USAGE_TYPE
  - `aws rds describe-db-instances --region ap-southeast-2`
  - `aws ec2 describe-snapshots --region ap-southeast-2 --owner-ids self`
  - `aws elasticache describe-cache-clusters --region ap-southeast-2`
- Prior investigation: `docs/investigations/2026-04-09-cost-quick-wins.md`

## Findings

### Monthly cost summary (USD)

| Service | Feb 2026 | Mar 2026 | Apr 2026 MTD (10 days) | Projected Apr full month |
|---|---:|---:|---:|---:|
| Vanta | — | 6,250.00 | — | — |
| EC2 Compute | 1,651.56 | 1,748.23 | 432.19 | ~1,400 |
| EC2 - Other | 1,331.61 | 1,380.32 | 406.98 | ~1,220 |
| CloudWatch | 1,033.70 | 1,159.77 | 309.70 | ~960 |
| RDS | 998.85 | 1,064.46 | 650.30 | ~670 |
| Tax | 578.88 | 634.33 | 222.74 | ~680 |
| End User Messaging (SMS) | 289.62 | 447.59 | 26.44 | ~80 |
| OpenVPN | 413.16 | 436.90 | 105.41 | ~320 |
| ElastiCache | 322.63 | 359.20 | 297.01 | ~310 |
| VPC | 271.43 | 308.24 | 88.50 | ~265 |
| ELB | 221.64 | 248.23 | 59.58 | ~180 |
| QuickSight | 102.00 | 102.00 | 31.17 | ~95 |

Notes:
- April projected full-month figures extrapolate from 10-day actuals; some services (RDS, ElastiCache) front-load Reserved Instance charges on day 1, so their projections are more reliable.
- Vanta has appeared as a ~6,250 USD lump sum in September 2025 and March 2026, suggesting a 6-month billing cadence. Next expected charge: September 2026.

### April 1 billing spike — explained

April 1 showed $1,175.87 vs the ~$190–200/day average for the rest of the month. This is **not an anomaly** — it is monthly Reserved Instance charges billed on the 1st:

| Driver | Apr 1 amount |
|---|---:|
| `APS2-HeavyUsage:db.t3.medium` (RDS reserved) | $480.82 |
| `APS2-HeavyUsage:cache.t3.medium` (ElastiCache reserved) | $293.76 |
| Tax | $222.74 |
| Everything else | ~$178 |
| **Total** | **~$1,175** |

The same pattern occurred in prior months but is more visible now that daily granularity is available. No investigation needed.

---

## New Finding 1: EBS Snapshot Accumulation — High Priority

### What was found

- **5,056 total snapshots** in `ap-southeast-2` as of 2026-04-10
- **153,256 GB of allocated snapshot volume** across those snapshots
- Oldest snapshot dates to **September 2021** — nearly 5 years old
- **4,747 snapshots (93.9%) are older than 90 days**
- **3,928 snapshots (77.7%) are older than 180 days**
- Only 97 snapshots were created in the last 30 days

### Snapshot creation history

The account had low snapshot activity until late 2024, then volume scaled sharply:

| Period | Snapshots created |
|---|---:|
| 2021–2024-10 (3 years) | ~168 |
| 2024-11 | 369 |
| 2024-12 | 327 |
| 2025-01 | 150 |
| 2025-02 | 182 |
| 2025-03 | 586 |
| 2025-04 | 536 |
| 2025-05 | 339 |
| 2025-06 | 358 |
| 2025-07 | 428 |
| 2025-08 | 236 |
| 2025-09 | 164 |
| 2025-10 | 567 |
| 2025-11 | 122 |
| 2025-12 | 121 |
| 2026-01 | 108 |
| 2026-02 | 109 |
| 2026-03 | 74 |
| 2026-04 (10 days) | 59 |

Likely cause: an automated snapshot policy was introduced around November 2024 and is creating regular snapshots, but there is no retention/deletion lifecycle policy. Old snapshots are accumulating indefinitely.

### Cost impact

EBS snapshot cost has grown steadily from $387/month in July 2025 to $631/month in March 2026:

| Month | Snapshot cost (USD) |
|---|---:|
| 2025-07 | 386.95 |
| 2025-08 | 508.25 |
| 2025-09 | 524.15 |
| 2025-10 | 554.47 |
| 2025-11 | 591.84 |
| 2025-12 | 602.66 |
| 2026-01 | 612.47 |
| 2026-02 | 622.10 |
| 2026-03 | 630.87 |
| 2026-04 MTD | 190.29 |

At current trajectory (~+$20/month growth), snapshot cost will reach $700+/month by Q4 2026 without intervention.

### Recommendation

Implement a Data Lifecycle Manager (DLM) policy that:
1. Retains the last 7 daily snapshots per volume
2. Retains the last 4 weekly snapshots per volume
3. Deletes everything older

Before running bulk deletion, confirm with the team whether any snapshots are required for compliance or disaster recovery beyond 90 days. If a 90-day retention policy is acceptable, deleting snapshots older than 90 days would eliminate ~4,747 snapshots (~93% of the current inventory). The actual billed storage reduction depends on deduplication, but with $631/month on ~153K allocated GB, even a 70% reduction in stored data would save ~$440/month.

**Savings estimate: $350–500 USD/month once lifecycle policy is in place and old snapshots are purged.**

---

## New Finding 2: `odoo-production` io1 → gp3 Migration — Easy Win

### What was found

Of the 8 RDS instances in `ap-southeast-2`, **only `odoo-production` is on `io1` storage**:

| Instance | Engine | Class | Storage | Type | IOPS | Multi-AZ |
|---|---|---|---|---|---|---|
| `sensor-prod` | MySQL | t3.medium | 20 GB | gp2 | — | Yes |
| `odoo-production` | PostgreSQL | t3.medium | 100 GB | **io1** | **3,000** | No |
| `sensor-qa` | MySQL | t3.medium | 50 GB | gp3 | 3,000 | No |
| `odoo-qa` | PostgreSQL | t3.medium | 20 GB | gp3 | 3,000 | No |
| `sensor-sandbox` | MySQL | t3.medium | 20 GB | gp2 | — | Yes |
| `smokealaramuat` | MySQL | t3.medium | 20 GB | gp2 | — | Yes |
| `odoo-dev` | PostgreSQL | t3.medium | 20 GB | gp3 | 3,000 | No |
| `dev-db-temp-mysql-8` | MySQL | t3.micro | 50 GB | gp2 | — | No |

`APS2-RDS:PIOPS` costs $330/month in March 2026 — and this single instance is the sole driver.

### Why this is safe to change

`gp3` volumes include 3,000 IOPS at no extra charge. Migrating `odoo-production` from `io1 3000 IOPS` to `gp3 3000 IOPS` achieves the same throughput at lower cost:

| | io1 (current) | gp3 (proposed) |
|---|---|---|
| Storage charge | $0.138/GB = $13.80/month | $0.122/GB = $12.20/month |
| IOPS charge | $0.11 × 3,000 = $330/month | Included |
| **Total** | **~$343.80/month** | **~$12.20/month** |

**Savings: ~$330 USD/month.**

The migration is an in-place RDS storage modification with a brief performance impact window. QA and dev are already on gp3 at 3,000 IOPS — this is a validated configuration.

This change requires written approval before execution as it is a production RDS modification.

### Side observation: non-prod Multi-AZ

`sensor-sandbox` and `smokealaramuat` both have `MultiAZ = true`. For non-production environments this doubles the instance cost. If these environments do not require HA, disabling Multi-AZ would save approximately:
- `APS2-Multi-AZUsage:db.t3.medium` = $96.72/month total in March
- Two fewer Multi-AZ instances would save roughly $60–65/month

---

## New Finding 3: SMS Drop ~85% Since March 26

### What was found

Daily SMS spend through `AWS End User Messaging` dropped sharply around March 26–27:

| Period | Daily avg |
|---|---:|
| Mar 1–25 | ~$16.20 |
| Mar 26–31 | ~$1.95 |
| Apr 1–10 | ~$2.64 |

March 1–25 was running at $16/day. From March 26 onwards it dropped to ~$2–4/day — an **85% reduction**.

### Interpretation

This is almost certainly a deliberate change (notification throttling, suppression fix, or routing change) made around March 26. It is worth confirming with the team:
- Was a notification batching or deduplication fix deployed on or around March 26?
- Is the current lower rate expected and stable, or is it a temporary suppression?

If the lower rate is the new normal, **April SMS will project to ~$75–90/month vs $437 in March** — a saving of ~$350/month that has already been captured without infrastructure changes.

If the drop is unintentional (a bug suppressing legitimate notifications), it should be investigated urgently from a product/reliability perspective.

---

## Updated EC2 - Other Breakdown

| Usage type | Mar 2026 | Apr 2026 MTD |
|---|---:|---:|
| EBS Snapshot | $630.87 | $190.29 |
| EBS gp2 volume hours | $288.60 | $84.87 |
| NAT Gateway hours | $219.48 | $64.96 |
| Regional data transfer | $155.77 | $42.66 |
| EBS gp3 volume hours | $59.54 | $17.84 |
| NAT Gateway bytes | $21.79 | $5.46 |

EBS Snapshot is now the **largest single component of EC2 - Other** at $631/month, ahead of NAT ($241) and EBS volumes ($348). The prior investigation highlighted NAT; snapshots are the bigger target.

---

## Updated CloudWatch Breakdown

| Usage type | Mar 2026 | Apr 2026 MTD |
|---|---:|---:|
| MetricStreamUsage | $579.74 | $162.12 |
| MetricMonitorUsage | $255.80 | $67.18 |
| DataProcessing-Bytes | $171.64 | $43.17 |
| VendedLog-Bytes | $58.11 | $12.12 |
| GMD-Metrics | $27.32 | $8.11 |
| AlarmMonitorUsage | $28.21 | $7.94 |
| TimedStorage-ByteHrs | $18.98 | $6.30 |
| VendedLog-Bytes-WAFLogs | $9.18 | $2.72 |

No change in structure vs the previous investigation. The metric stream to New Relic remains the top driver.

---

## Updated Quick Wins Table

| # | Area | Mar 2026 baseline | New evidence | Updated savings band |
|---|---|---:|---|---:|
| 1 | EBS Snapshot lifecycle policy | $631/month **and growing** | 5,056 snapshots, oldest from 2021; 93% older than 90 days; no lifecycle policy visible | $350–500/month |
| 2 | CloudWatch metric stream scope | $1,160/month | Unchanged — NewRelic-Metric-Stream still running | $300–700/month |
| 3 | `odoo-production` io1 → gp3 | $330/month PIOPS | Confirmed sole io1 instance; QA/dev already on gp3 | ~$330/month |
| 4 | NAT gateways | $241/month | Unchanged | $50–200/month |
| 5 | RDS non-prod rightsizing/Multi-AZ | $1,064/month total | Sandbox and UAT have unnecessary Multi-AZ; PIOPS is separate (item 3) | $60–100/month |
| 6 | Redis non-prod rightsizing | $359/month | 6 t3.medium + 4 t3.micro still running; no scheduling observed | $50–180/month |
| 7 | SMS | $437/month in Mar | **Already dropped ~85% as of Mar 26** — confirm intent | $0 (already captured?) |
| 8 | OpenVPN licensing | $437/month | Unchanged | $100–350/month |

**Revised total savings estimate: $890–$2,360/month from items 1–6 (SMS savings likely already realised).**

---

## Risks Or Constraints

- EBS snapshot deletion should not proceed without confirming retention requirements with the team. Some snapshots may be compliance artefacts.
- The `odoo-production` io1 → gp3 migration is a live production RDS change. It requires explicit written approval and a maintenance window.
- The SMS drop is unexplained — if it reflects suppressed notifications rather than a product change, reliability may be degraded without a visible alarm.
- All April figures are 10-day MTD. Projections could shift if usage patterns change mid-month.

## Recommended Next Steps

1. **Confirm SMS drop intent** with whoever owns the notification stack — was this a deliberate throttle/dedup or a silent failure?
2. **Investigate snapshot lifecycle**: identify which automated policy creates snapshots, confirm retention requirements, propose a DLM policy.
3. **Propose `odoo-production` io1 → gp3 change** as a formal written approval request — it is the clearest single-instance win.
4. **Confirm sandbox/UAT Multi-AZ** with owners — if not needed, disable for immediate savings.
5. **Pull CloudWatch metric stream config** to enumerate which namespaces are exported to New Relic and assess filtering opportunities.

## Open Questions

- What triggered the snapshot automation ramp-up in November 2024?
- Is there a compliance or DR policy that mandates retention beyond 90 days?
- Was the SMS volume reduction on March 26 intentional?
- Are the `sensor-sandbox` and `smokealaramuat` Multi-AZ configurations intentional or inherited defaults?
- Has the `odoo-production` io1 configuration ever been reviewed, or was it set at creation and not revisited?
