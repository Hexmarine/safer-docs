# Cost Quick Wins Baseline

- Date: 2026-04-09
- Scope: quick-win cost optimization opportunities across AWS infrastructure, messaging, marketplace, and vendor charges
- Related systems: AWS Cost Explorer, CloudWatch, VPC/NAT, EC2, RDS, ElastiCache, SMS, OpenVPN marketplace, Vanta

## Objective

Identify the fastest evidence-backed cost optimization opportunities in the current AWS account and separate them into infrastructure, product-usage, and vendor-review workstreams.

## Sources Used

- Repository files:
  - `AGENTS.md`
  - `docs/investigations/2026-04-09-aws-account-baseline.md`
  - `docs/investigations/2026-04-09-ap-southeast-2-architecture-baseline.md`
  - `docs/infra/ap-southeast-2-architecture.md`
- AWS commands or consoles:
  - `aws ce get-cost-and-usage --time-period Start=2026-03-01,End=2026-04-01 --granularity MONTHLY --metrics UnblendedCost --group-by Type=DIMENSION,Key=SERVICE`
  - `aws ce get-cost-and-usage --time-period Start=2026-04-01,End=2026-04-10 --granularity MONTHLY --metrics UnblendedCost --group-by Type=DIMENSION,Key=SERVICE`
  - `aws ce get-cost-and-usage` filtered by `SERVICE` and grouped by `USAGE_TYPE` for:
    - `EC2 - Other`
    - `AmazonCloudWatch`
    - `Amazon Relational Database Service`
    - `Amazon ElastiCache`
    - `AWS End User Messaging`
    - `OpenVPN Access Server (10 Connected Devices) / Self-Hosted VPN`
    - `Vanta`
  - `aws cloudwatch list-metric-streams --region ap-southeast-2`
  - `aws ec2 describe-nat-gateways --region ap-southeast-2`
  - `aws ec2 describe-addresses --region ap-southeast-2`
  - `aws elasticache describe-cache-clusters --region ap-southeast-2 --show-cache-node-info`
- External references: none

## Findings

- March 2026 is the latest complete month and is the right baseline for this note.
- April 1-9, 2026 month-to-date costs broadly track the same major spend categories as March 2026.
- For AUD estimates in this note, `1 USD = 1.4140 AUD` was used, derived from the RBA published `AUD/USD = 0.7072` as at `8 April 2026`.
- Ignoring `Tax` and the non-infra vendor charge `Vanta`, the biggest March 2026 cost drivers were:
  - `Amazon Elastic Compute Cloud - Compute`: `1748.23 USD`
  - `EC2 - Other`: `1380.32 USD`
  - `AmazonCloudWatch`: `1159.77 USD`
  - `Amazon Relational Database Service`: `1064.46 USD`
  - `AWS End User Messaging`: `447.59 USD`
  - `OpenVPN Access Server (10 Connected Devices) / Self-Hosted VPN`: `436.90 USD`
  - `Amazon ElastiCache`: `359.20 USD`
  - `Amazon Virtual Private Cloud`: `308.24 USD`
  - `Amazon Elastic Load Balancing`: `248.23 USD`
- `Vanta` appears as a separate `6250 USD` contract-style charge and does not look like a normal AWS resource optimization target.

## Quick Wins

| Rank | Cost area | March 2026 baseline | Evidence | Likely savings mechanism | Rough savings band | Tradeoffs or blockers | Next safe validation step |
| --- | --- | ---: | --- | --- | --- | --- | --- |
| 1 | CloudWatch metric streams and monitoring volume | `1159.77 USD` | `APS2-CW:MetricStreamUsage = 579.74 USD`, `APS2-DataProcessing-Bytes = 171.64 USD`, `APS2-CW:MetricMonitorUsage = 255.80 USD`; one running metric stream `NewRelic-Metric-Stream` | Reduce or filter exported metrics, confirm whether full-fidelity streaming to New Relic is still required, trim unnecessary custom/monitored metrics | `300-700 USD` per month if metric stream scope is materially reduced | Observability loss if filters are too aggressive; requires coordination with whoever consumes New Relic metrics | Inspect metric stream configuration, Firehose delivery settings, and the monitoring requirements for teams relying on New Relic |
| 2 | NAT gateways across five environment VPCs | `219.48 USD` NAT hours plus `21.79 USD` NAT bytes inside `EC2 - Other`; VPC service total `308.24 USD` | Five active NAT gateways, one per non-default VPC | Remove underused non-prod NAT gateways, replace some egress paths with VPC endpoints, or consolidate environments where safe | `50-200 USD` per month near term, more if endpoints or environment cleanup reduce transfer | NAT changes can break private-subnet outbound access and patching if not validated carefully | Measure NAT traffic per gateway and correlate each gateway to active non-prod workloads before proposing any topology change |
| 3 | RDS provisioned IOPS and non-prod sizing | `1064.46 USD` | `APS2-RDS:PIOPS = 330 USD`, `APS2-HeavyUsage:db.t3.medium = 496.84 USD`, `APS2-Multi-AZUsage:db.t3.medium = 96.72 USD`; multiple non-prod DBs exist across `dev`, `qa`, `sandbox`, and `uat` | Reduce provisioned IOPS where over-provisioned, rightsize non-prod DB classes, review whether all non-prod DBs need to be always-on | `150-400 USD` per month if PIOPS and non-prod footprint are oversized | Database performance risk and environment fidelity concerns; production changes need stronger evidence than non-prod | Pull per-instance storage, IOPS, and performance metrics to identify which DBs drive the PIOPS and whether non-prod instances are oversized |
| 4 | Redis duplication across environments | `359.20 USD` | Ten Redis nodes in `ap-southeast-2`: four `cache.t3.micro` in `dev/qa` and six `cache.t3.medium` in `sandbox/prod/uat`; billing shows `303.55 USD` for `cache.t3.medium` and `55.65 USD` for `cache.t3.micro` | Rightsize or schedule non-prod Redis, remove unused replicas or duplicate clusters, review whether `uat` and `sandbox` both need medium-sized pairs | `50-180 USD` per month depending on redundancy and environment usage | Cache topology may be tied to app expectations or HA testing; shutting down caches can disrupt environments | Map each Redis cluster to the owning application path and confirm whether non-prod pairs are required for HA or can be reduced |
| 5 | Australian outbound SMS fees | `437.64 USD` out of `447.59 USD` total `AWS End User Messaging` | Cost Explorer shows nearly all spend in `APS2-OutboundSMS-AU-Standard-Senderid-MessageFees` | Reduce message volume, improve alert routing, deduplicate notifications, or move some traffic to lower-cost channels where product allows | `50-250 USD` per month depending on message necessity | This is product and customer-communication policy, not pure infra tuning; delivery/compliance needs may limit changes | Break SMS costs down by use case or originating workflow before recommending rate or routing changes |
| 6 | OpenVPN marketplace licensing | `436.90 USD` | Cost Explorer shows `APS2-SoftwareUsage:t2.micro = 436.896 USD`; architecture inventory shows multiple environment OpenVPN instances across `prod`, `qa`, `sandbox`, `uat`, and a stopped `dev` host | Reduce the number of licensed environments, replace low-value non-prod VPNs, or centralize access patterns | `100-350 USD` per month if some environments no longer need dedicated VPN licensing | Access workflows may depend on per-environment VPN isolation; licensing model should be confirmed before assuming per-instance savings | Confirm how the marketplace fee maps to deployed VPN instances and whether separate environment VPN endpoints are still required |
| 7 | Elastic IP and public IP sprawl | not directly isolated, but likely contributes to `EC2 - Other` and operational overhead | Large Elastic IP inventory with several addresses unattached to instances and some completely unassociated; many public IPs across non-prod | Release unused EIPs and reduce unnecessary public addressing on non-prod instances | `low` direct savings, moderate governance/security value | Savings may be smaller than the operational cleanup value; some attached ENIs may be NAT or LB related | Classify all Elastic IPs into active app use, NAT/LB use, and orphaned allocations before recommending cleanup |
| 8 | Vendor contract review: Vanta | `6250 USD` | Cost Explorer shows `Global-SoftwareUsage-Contracts = 6250` under `Vanta` | Procurement review, contract right-sizing, or licensing review | potentially large, but external to AWS infra optimization | Not actionable by infra changes alone; likely requires vendor owner and procurement context | Confirm billing cadence, contract scope, and business owner before treating this as a savings program |

## AUD Equivalents

- CloudWatch metric export scope:
  - current monthly baseline: about `1639.95 AUD`
  - likely savings band: about `424.21 AUD` to `989.82 AUD` per month
- NAT gateways:
  - current monthly baseline for identified NAT hour and byte charges: about `341.16 AUD`
  - likely savings band: about `70.70 AUD` to `282.81 AUD` per month
- RDS rightsizing and provisioned IOPS:
  - current monthly baseline: about `1505.18 AUD`
  - likely savings band: about `212.10 AUD` to `565.61 AUD` per month
- Redis duplication:
  - current monthly baseline: about `507.92 AUD`
  - likely savings band: about `70.70 AUD` to `254.52 AUD` per month
- SMS usage:
  - current monthly baseline: about `632.90 AUD`
  - likely savings band: about `70.70 AUD` to `353.51 AUD` per month
- OpenVPN licensing:
  - current monthly baseline: about `617.79 AUD`
  - likely savings band: about `141.40 AUD` to `494.91 AUD` per month
- Elastic IP and public IP sprawl:
  - current evidence is not strong enough to estimate a reliable AUD savings band
  - likely direct savings are smaller than the items above unless many idle public IPv4 allocations are confirmed
- Vanta contract review:
  - current monthly baseline: about `8837.67 AUD`
  - no engineering-led savings band is estimated here because this appears to be procurement or contract work rather than AWS resource optimization

## AUD Roll-Up

- Actionable low-hanging fruit subtotal, excluding Elastic IP cleanup and excluding Vanta:
  - current monthly baseline: about `5244.90 AUD`
  - total likely savings band: about `989.81 AUD` to `2941.18 AUD` per month
- Broader subtotal including Vanta baseline, but still excluding Elastic IP cleanup:
  - current monthly baseline: about `14082.57 AUD`
  - total likely savings band remains `989.81 AUD` to `2941.18 AUD` per month from the engineering-led items above
  - additional Vanta savings are possible, but they require vendor or procurement review and are not estimated here
- Elastic IP and public IP cleanup remain outside the rolled-up savings total because the current evidence is not strong enough to estimate a reliable AUD range.

## 12-Month Expenditure Trend In AUD

| Billing window | Total spend | Top 5 contributors | Potential savings |
| --- | ---: | --- | --- |
| `2025-04-01` to `2025-05-01` | `A$0.00` | no billable spend returned by Cost Explorer | `A$0` |
| `2025-05-01` to `2025-06-01` | `A$0.00` | no billable spend returned by Cost Explorer | `A$0` |
| `2025-06-01` to `2025-07-01` | `A$0.00` | no billable spend returned by Cost Explorer | `A$0` |
| `2025-07-01` to `2025-08-01` | `A$8,845.07` | `EC2 Compute A$2,054.90`; `EC2 - Other A$1,386.74`; `RDS A$1,199.81`; `CloudWatch A$1,106.56`; `Tax A$674.15` | `A$990-A$2,941` from recurring infra and usage quick wins |
| `2025-08-01` to `2025-09-01` | `A$10,977.27` | `EC2 Compute A$2,549.22`; `EC2 - Other A$1,764.12`; `RDS A$1,502.00`; `CloudWatch A$1,414.67`; `Tax A$823.16` | `A$990-A$2,941` from recurring infra and usage quick wins |
| `2025-09-01` to `2025-10-01` | `A$19,641.41` | `Vanta A$8,837.67`; `EC2 Compute A$2,478.26`; `EC2 - Other A$1,771.91`; `RDS A$1,471.47`; `CloudWatch A$1,404.19` | `A$990-A$2,941` from engineering-led quick wins, plus additional unestimated Vanta review potential |
| `2025-10-01` to `2025-11-01` | `A$11,644.56` | `EC2 Compute A$2,577.55`; `EC2 - Other A$1,845.15`; `RDS A$1,515.99`; `CloudWatch A$1,484.43`; `Tax A$890.29` | `A$990-A$2,941` from recurring infra and usage quick wins |
| `2025-11-01` to `2025-12-01` | `A$11,176.29` | `EC2 Compute A$2,484.60`; `EC2 - Other A$1,874.99`; `CloudWatch A$1,504.41`; `RDS A$1,485.41`; `Tax A$853.07` | `A$990-A$2,941` from recurring infra and usage quick wins |
| `2025-12-01` to `2026-01-01` | `A$11,369.95` | `EC2 Compute A$2,556.35`; `EC2 - Other A$1,909.40`; `RDS A$1,503.54`; `CloudWatch A$1,475.23`; `Tax A$867.01` | `A$990-A$2,941` from recurring infra and usage quick wins |
| `2026-01-01` to `2026-02-01` | `A$11,619.94` | `EC2 Compute A$2,578.51`; `EC2 - Other A$1,929.49`; `RDS A$1,504.71`; `CloudWatch A$1,500.78`; `Tax A$887.56` | `A$990-A$2,941` from recurring infra and usage quick wins |
| `2026-02-01` to `2026-03-01` | `A$10,707.85` | `EC2 Compute A$2,335.35`; `EC2 - Other A$1,882.93`; `CloudWatch A$1,461.68`; `RDS A$1,412.40`; `Tax A$818.55` | `A$990-A$2,941` from recurring infra and usage quick wins |
| `2026-03-01` to `2026-04-01` | `A$20,548.99` | `Vanta A$8,837.67`; `EC2 Compute A$2,472.04`; `EC2 - Other A$1,951.80`; `CloudWatch A$1,639.94`; `RDS A$1,505.18`; `Tax A$897.10`; `End User Messaging A$632.89`; `OpenVPN A$617.79`; `ElastiCache A$507.92`; `VPC A$435.90` | `A$990-A$2,941` from engineering-led quick wins, plus additional unestimated Vanta review potential |

### Notes On The Potential Savings Column

- The `A$990-A$2,941` range is the current low-hanging-fruit savings band already derived for recurring engineering-led items:
  - CloudWatch and New Relic scope reduction
  - NAT gateway and network cleanup
  - RDS rightsizing and PIOPS review
  - Redis duplication cleanup
  - SMS usage controls
  - OpenVPN licensing review
- The table does not allocate that range precisely per month because the supporting deep-dive evidence was collected against the current account shape rather than reconstructed month by month.
- `Vanta` is called out separately where it appears because it looks like a vendor or procurement review item, not a normal AWS infrastructure optimization lever.

## March 2026 Detailed Breakdown In AUD

This breakdown expands the `2026-03-01` to `2026-04-01` row so the month more closely ties back to the total of `A$20,548.99`.

| Contributor | March 2026 spend |
| --- | ---: |
| `Vanta` | `A$8,837.67` |
| `Amazon Elastic Compute Cloud - Compute` | `A$2,472.04` |
| `EC2 - Other` | `A$1,951.80` |
| `AmazonCloudWatch` | `A$1,639.94` |
| `Amazon Relational Database Service` | `A$1,505.18` |
| `Tax` | `A$896.96` |
| `AWS End User Messaging` | `A$632.90` |
| `OpenVPN Access Server (10 Connected Devices) / Self-Hosted VPN` | `A$617.78` |
| `Amazon ElastiCache` | `A$507.92` |
| `Amazon Virtual Private Cloud` | `A$435.85` |
| `Amazon Elastic Load Balancing` | `A$351.00` |
| `Amazon QuickSight` | `A$144.23` |
| `Amazon Inspector` | `A$99.48` |
| `AWS Security Hub` | `A$90.37` |
| `AWS WAF` | `A$88.95` |
| `Other minor services combined` | `A$276.90` |
| **Total** | **`A$20,548.99`** |

The `Other minor services combined` line includes the remaining smaller March contributors such as `AWS Config`, `Amazon S3`, `GuardDuty`, `Global Accelerator`, `Macie`, `Systems Manager`, `Route 53`, `KMS`, `Secrets Manager`, `Firehose`, `Glue`, and several near-zero services.

## Service Breakdown Notes

### EC2 and network

- `EC2 - Other` is not mainly data transfer in this account. The clearest high-value components are:
  - `APS2-NatGateway-Hours = 219.48 USD`
  - `APS2-NatGateway-Bytes = 21.79 USD`
  - smaller amounts of VPC peering, cross-region transfer, snapshots, and CPU credits
- This means the fastest network-cost review should focus on NAT topology and public IP hygiene rather than generic data-transfer tuning.

### EC2 compute breakdown

- March 2026 `Amazon Elastic Compute Cloud - Compute` spend was `1748.23 USD`, about `2472.04 AUD`.
- Current running instance count in `ap-southeast-2` is:
  - `15` x `t3.medium`
  - `8` x `t3.small`
  - `6` x `t2.micro`
  - `3` x `t2.medium`
  - `2` x `t3.large`
  - `1` x `c5.xlarge`
  - `1` x `c5.large`
- March 2026 EC2 compute cost by usage type was:
  - `t3.medium`: `798.17 USD`, about `1128.51 AUD`
  - `t3.large`: `177.66 USD`, about `251.27 AUD`
  - `t3.small`: `176.76 USD`, about `249.99 AUD`
  - `c5.large`: `170.53 USD`, about `241.17 AUD`
  - `c5.xlarge`: `165.17 USD`, about `233.58 AUD`
  - `t2.medium`: `130.33 USD`, about `184.29 AUD`
  - `t2.micro`: `84.36 USD`, about `119.29 AUD`
  - `t2.small`: `21.73 USD`, about `30.72 AUD`
  - EC2 data transfer inside the compute service bucket: `23.53 USD`, about `33.28 AUD`
- The key takeaway is that the visibly larger instances are not the main EC2 compute cost driver.
- The bigger cost lever is the large count of always-on `t3.medium` instances, especially the repeated API and Odoo-style hosts across `prod`, `qa`, `sandbox`, and `uat`.
- The `c5` instances are still worth review because they are larger and specialized:
  - running `c5.xlarge`: `Emqx-prod`
  - running `c5.large`: `Emqx-sandbox`
  - stopped `c5.large` instances also exist in `qa` and `uat`
- If the goal is quick savings, the first EC2 review should focus on:
  - whether all `t3.medium` non-prod instances need to be always on
  - whether duplicate API and Odoo hosts in `qa`, `sandbox`, and `uat` can be scheduled or consolidated
  - whether the EMQX `c5` sizing is still justified by actual workload

#### `t3.medium` instance assessment

| Instance or group | Likely environment | Likely role | State | Savings likelihood |
| --- | --- | --- | --- | --- |
| `prod-api-server` x5 | prod | core backend API fleet | running | low |
| `prod-kafka` | prod | messaging / event pipeline | running | low |
| `wordpress-prod` | prod | public web / CMS | running | low |
| `sandbox-api-server` x2 | sandbox | non-prod backend API fleet | running | high |
| `uat-api-server` | uat | non-prod backend API | running | high |
| `Odoo-sandbox` | sandbox | non-prod ERP / back office | running | medium |
| `odoo-qa-private-v2` | qa | non-prod ERP / back office | running | medium |
| `odoo-qa-may-5` | qa | likely older QA Odoo variant | running | high |
| `odoo-qa-latest` | qa | current QA Odoo variant | running | medium |
| `SANDBOX-Kafka-mongo-Smoke` | sandbox | non-prod messaging / data services | running | medium |
| `Qa-kafka` | qa | non-prod messaging | stopped | high |
| `Qa-kafka-private` | qa | non-prod messaging | stopped | high |
| `Dev-Kafka` | dev | non-prod messaging | stopped | high |
| `uat-kafka` | uat | non-prod messaging | stopped | high |
| `sonarqube-dev` | dev | developer tooling | stopped | high |
| `odoo-uat-server` | uat | non-prod ERP / back office | stopped | high |
| `Odoo-development-private` | dev | non-prod ERP / back office | stopped | high |
| `Odoo-development-private-v2` | dev | likely newer dev Odoo variant | stopped | high |
| `odoo-qa-private` | qa | non-prod ERP / back office | stopped | high |
| `dev-api-server` x2 | dev | non-prod backend API | stopped | high |

- The strongest current EC2 quick-win signal is not one oversized machine. It is the repeated `t3.medium` pattern across `qa`, `sandbox`, `uat`, and `dev`.
- The highest-likelihood cleanup candidates are the stopped `t3.medium` fleet plus apparently duplicated QA Odoo variants such as `odoo-qa-may-5`, `odoo-qa-latest`, and `odoo-qa-private`.
- The highest-likelihood running-cost reduction candidates are `sandbox-api-server` x2 and `uat-api-server`, if those environments do not need continuous uptime.

### CloudWatch

- CloudWatch spend is unusually high relative to the size of the visible workload footprint.
- The dominant drivers are:
  - `APS2-CW:MetricStreamUsage = 579.74 USD`
  - `APS2-CW:MetricMonitorUsage = 255.80 USD`
  - `APS2-DataProcessing-Bytes = 171.64 USD`
  - `APS2-VendedLog-Bytes = 58.11 USD`
- A running metric stream named `NewRelic-Metric-Stream` strongly suggests third-party observability export is a primary cost lever.

### RDS

- RDS spend is driven more by provisioned performance than storage:
  - `APS2-RDS:PIOPS = 330 USD`
  - `APS2-HeavyUsage:db.t3.medium = 496.84 USD`
  - `APS2-Multi-AZUsage:db.t3.medium = 96.72 USD`
- The architecture baseline already shows multiple non-prod databases across separate VPCs, so non-prod footprint review is likely worthwhile.

### ElastiCache

- Cache spend is simple and visible:
  - `APS2-HeavyUsage:cache.t3.medium = 303.55 USD`
  - `APS2-NodeUsage:cache.t3.micro = 55.65 USD`
- The account has ten Redis cache nodes, with duplicated environment pairs in `dev`, `qa`, `sandbox`, `prod`, and `uat`.

### Messaging and vendor costs

- `AWS End User Messaging` is primarily a product-usage cost, not a core infrastructure inefficiency.
- `OpenVPN` appears to be marketplace software usage rather than EC2 compute itself.
- `Vanta` looks contractual and should be handled as vendor/procurement review.

## Risks Or Constraints

- This note is optimized for quick wins, not full chargeback accuracy.
- Cost Explorer service labels do not always map one-to-one to a single resource, so some recommendations remain inference until resource-level validation is completed.
- Marketplace and vendor charges may not be reducible by engineering changes alone.
- NAT, observability, and database changes can all create production risk if implemented without dependency analysis.

## Cost Optimization Opportunities

- Prioritize CloudWatch metric stream scope review first because it has a large monthly cost and a clear concrete resource to inspect.
- Prioritize NAT gateway traffic validation second because the environment layout suggests possible non-prod overprovisioning.
- Prioritize RDS PIOPS and non-prod database sizing third because the cost is significant and the architecture already shows duplicated environment footprint.
- Treat SMS, OpenVPN, and Vanta as important but differently owned:
  - SMS belongs to product and notification policy review
  - OpenVPN belongs to access architecture and licensing review
  - Vanta belongs to procurement/vendor review

## Recommended Next Steps

- Deep dive note created: `docs/investigations/2026-04-09-cloudwatch-newrelic-deep-dive.md`
- Create a follow-up investigation that measures NAT gateway bytes and per-environment outbound dependency.
- Inventory RDS storage, IOPS, and performance metrics per instance to identify likely overprovisioning.
- Map Redis clusters and OpenVPN instances to active environment owners and required use cases.
- Identify the business workflows generating Australian SMS traffic before recommending reductions.
- Confirm the owner and billing intent for the Vanta contract line item outside the infra workstream.

## Open Questions

- Is the `NewRelic-Metric-Stream` still required at its current breadth, and who owns it?
- Which non-production environments are still operationally necessary as always-on replicas of production?
- Which RDS instances consume the provisioned IOPS line item, and are those settings justified by actual load?
- Are all OpenVPN endpoints actively used, or are some legacy access paths?
- What customer or internal workflows generate the Australian SMS charges?
