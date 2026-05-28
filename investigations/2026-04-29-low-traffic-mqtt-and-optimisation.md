# Low-Traffic MQTT And Optimisation Check

- Date: 2026-04-29
- Scope: read-only check for MQTT activity and low-traffic cost optimisation candidates ahead of an expected quiet month
- Related systems: EMQX, EC2, ALB, NAT Gateway, RDS, ElastiCache, EBS, Elastic IPs, Terraform baseline

## Objective

Determine whether anything is still communicating with the production MQTT broker, then identify safe optimisation candidates for a low-traffic month without executing any AWS changes.

## Sources Used

- Repository files:
  - `docs/investigations/2026-04-09-mqtt-device-communications.md`
  - `docs/infra/prod-services-access.md`
  - `docs/investigations/2026-04-10-prod-user-device-migration-readiness.md`
  - `docs/investigations/2026-04-11-cost-optimization-proposals.md`
  - `docs/investigations/2026-04-20-infra-inventory.md`
  - `docs/investigations/2026-04-29-terraform-drift-status.md`
- AWS read-only commands:
  - `aws ec2 describe-instances`
  - `aws ec2 describe-addresses`
  - `aws ec2 describe-nat-gateways`
  - `aws rds describe-db-instances`
  - `aws elasticache describe-replication-groups`
  - `aws elbv2 describe-load-balancers`
  - `aws lambda list-functions`
  - `aws cloudwatch get-metric-statistics`
  - `aws cloudwatch list-metrics`
  - `aws ce get-cost-and-usage`

## Findings

### MQTT activity

Production EMQX is still receiving continuous network traffic.

Observed production broker:

| Item | Value |
|---|---|
| Instance | `i-0b381a2bfdf65a329` |
| Name | `Emqx-prod` |
| Instance type | `c5.xlarge` |
| State | `running` |
| Public IP | `13.211.102.112` |
| Private IP | `10.0.2.98` |
| Public DNS | `mqtt.sensorglobal.com` |

CloudWatch host metrics for `2026-04-22T00:00:00Z` to `2026-04-29T00:00:00Z` show:

| Metric | 7-day observation |
|---|---:|
| `NetworkIn` | 178,444,689 bytes total, 1,081,483 bytes/hour average, 617,797 bytes/hour minimum |
| `NetworkOut` | 100,897,765 bytes total, 611,502 bytes/hour average |
| `NetworkPacketsIn` | 662,124 packets total, 4,013 packets/hour average |
| `CPUUtilization` | 0.318% average, 4.853% maximum |
| `CWAgent mem_used_percent` | 5.987% average, 6.567% maximum |

Interpretation: something is continuously communicating with the broker host, with no quiet hour in the measured week. The traffic level is low and consistent with MQTT keep-alives and/or low-volume device messages.

Important caveat: these are EC2 host-level metrics. They do not prove the exact MQTT client count, port, topics, or message types. Exact MQTT visibility requires EMQX-level access such as the dashboard or `emqx_ctl clients list`, or a VPC Flow Logs / packet-level investigation.

Sandbox EMQX is stopped:

| Item | Value |
|---|---|
| Instance | `i-07af1c06915741068` |
| Name | `Emqx-sandbox` |
| Instance type | `c5.large` |
| State | `stopped` |

### Load balancer activity

Corrected CloudWatch dimensions show production ALBs are low traffic but not dead. Non-prod and sandbox ALBs observed below had no request datapoints in the same 7-day window.

| Load balancer | 7-day requests | Max hourly requests |
|---|---:|---:|
| `SmokeAPI-LB` | 1,391,253 | 9,132 |
| `keychain-production-ALB` | 95 | 3 |
| `sso-api-lb` | 44,317 | 15,269 |
| `wordpress-prod-alb` | 68,119 | 10,238 |
| `wordpress-sensorinsure` | 32,189 | 1,362 |
| `Smoke-Sandbox-Api-LB` | 0 | 0 |
| `sandbox-sso-api` | 0 | 0 |
| `smoke-uat-api` | 0 | 0 |
| `SmokeqaLB` | 0 | 0 |
| `sso-dev-api-lb` | 0 | 0 |
| `qa-sso-api` | 0 | 0 |
| `wordpress-qa` | 0 | 0 |

### NAT Gateway activity

| NAT Gateway | Environment | 7-day traffic |
|---|---|---:|
| `nat-0419aece84205b4f5` | prod | 11,702,550,846 bytes in/out, 21,676,319 packets |
| `nat-0f00d13d55f11c828` | sandbox | 0 bytes, 0 packets |

The sandbox NAT Gateway had no measured traffic for the week but still incurs fixed hourly cost while available.

### RDS activity

CloudWatch metrics for the same 7-day window:

| DB instance | Shape | Key observation |
|---|---|---|
| `sensor-prod` | `db.t3.medium`, MySQL, Multi-AZ, gp2, 20 GB | 4.562% average CPU, 24.989% max CPU, 3.618 average connections, 65 max connections |
| `odoo-production` | `db.t3.medium`, PostgreSQL, Single-AZ, io1, 100 GB, 3,000 IOPS | 6.347% average CPU, 52.567% max CPU, 19.344 average connections |
| `sensor-sandbox` | `db.t3.medium`, MySQL, Multi-AZ, gp2, 20 GB | 3.958% average CPU, 7.208% max CPU, 0 average/max connections |

`sensor-sandbox` appears unused over the measured week and remains a strong optimisation candidate if sandbox can stay offline or be recreated from backup.

### ElastiCache activity

CloudWatch metrics for the same 7-day window:

| Node | Cluster | CPU average | Memory average | Connections average |
|---|---|---:|---:|---:|
| `smokeprodapiredis-001` | prod | 2.419% | 1.511% | 16.280 |
| `smokeprodapiredis-002` | prod | 2.180% | 1.519% | 5.837 |
| `smokealarmsandbox-001` | sandbox | 2.173% | 0.506% | 5.677 |
| `smokealarmsandbox-002` | sandbox | 2.366% | 0.505% | 4.641 |

Both prod and sandbox Redis clusters are lightly used by CPU and memory. Sandbox Redis is especially low-utilisation and should be considered with the wider sandbox suspension/decommission decision.

### Other inventory signals

- 8 Elastic IPs are idle/unassociated.
- 24 EBS volumes are unattached, totalling 745 GB.
- Several stopped sandbox/support instances remain present, including `Power-BI-Gateway`, `Odoo-sandbox`, `SANDBOX-Kafka-mongo-Smoke`, `openvpn-sandbox`, and sandbox cron/API components.
- Cost Explorer returned no usable grouped cost data for the queried April 2026 period in this session, so recommendations below rely on resource inventory and CloudWatch utilisation rather than fresh Cost Explorer totals.

## Risks Or Constraints

- AWS changes were not executed as part of this investigation.
- EMQX client counts were not queried directly because that requires broker-level access via dashboard or instance shell.
- EC2 network metrics cannot distinguish MQTT traffic from management traffic on the same host.
- Production MQTT is active, even if low-volume. Stopping EMQX is not safe without confirming all device and backend dependencies.
- RDS stop is only temporary; AWS automatically restarts stopped RDS instances after the maximum stop window. Durable savings require downsizing, deletion after snapshot, or environment retirement.
- Deleting EBS volumes, releasing EIPs, deleting ALBs, deleting NAT Gateways, and resizing databases/cache nodes are all infrastructure mutations and require explicit approval plus rollback planning.

## Cost Optimization Opportunities

### 1. Keep prod EMQX running, but right-size it after broker-level validation

Evidence: prod EMQX traffic is continuous but light, with 0.318% average CPU and 5.987% average memory on a `c5.xlarge`.

Candidate action: confirm connected client count and session/message load from EMQX, then test a smaller broker size during a maintenance window.

Risk: medium. MQTT is production-critical and active. Do not stop the broker for cost savings.

### 2. Remove or suspend inactive sandbox fixed-cost infrastructure

Evidence: sandbox EC2 app tier is stopped, sandbox EMQX is stopped, sandbox ALBs show 0 requests, sandbox NAT Gateway shows 0 bytes/packets, and `sensor-sandbox` has 0 DB connections.

Candidate action: if sandbox is not needed next month, take final snapshots/backups and delete or suspend:

- `nat-0f00d13d55f11c828`
- `Smoke-Sandbox-Api-LB`
- `sandbox-sso-api`
- `sensor-sandbox`
- `smokealarmsandbox`
- stopped sandbox EC2 instances and their attached storage where no longer needed

Risk: medium. Savings are attractive because these are fixed-cost resources, but recovery expectations need to be explicit before deletion.

### 3. Migrate `odoo-production` storage from io1 to gp3

Evidence from prior RDS deep analysis: `odoo-production` has 3,000 provisioned IOPS on io1 with documented low IOPS utilisation. The current 7-day check also shows low average CPU and stable low connection count.

Candidate action: perform an in-place RDS storage-type change from io1 to gp3 during a maintenance window.

Risk: low to medium. This is a production RDS modification and may cause a storage optimisation period, but the previous IOPS analysis showed gp3 has sufficient headroom.

### 4. Clean up unattached EBS volumes

Evidence: 24 available/unattached EBS volumes totalling 745 GB.

Candidate action: tag/owner-confirm, snapshot any volumes with uncertain recovery value, then delete confirmed obsolete volumes.

Risk: low to medium. Unattached volumes are usually safe cleanup candidates, but old manually managed environments can hide rollback dependencies.

### 5. Release idle Elastic IPs after ownership confirmation

Evidence: 8 unassociated EIPs.

Candidate action: map each EIP against DNS, historical docs, and decommission records, then release confirmed obsolete addresses.

Risk: low if DNS and owner checks are complete; high if released prematurely because public IPs cannot be reliably recovered.

### 6. Right-size Redis after dependency check

Evidence: prod Redis nodes average about 1.5% memory use; sandbox Redis nodes average about 0.5% memory use.

Candidate action: delete sandbox Redis if sandbox is suspended; consider smaller prod Redis node class only after validating memory headroom, failover expectations, and latency sensitivity.

Risk: medium for prod, lower for sandbox if sandbox remains offline.

### 7. Prod API autoscaling is already improved

Evidence: Terraform and live AWS now agree on prod API ASG min 2, desired 3, max 5, with stopped warm-pool instances.

Candidate action: no immediate further action unless expected traffic drops enough to justify an approved temporary min/desired reduction. Because the service is production-facing, keep at least two healthy instances unless the team explicitly accepts lower availability.

Risk: low if left as-is.

## Recommended Next Steps

1. For MQTT, get broker-level evidence before any EMQX right-sizing:
   - connected client count
   - client IP distribution
   - subscriptions/topic activity
   - retained/session counts
   - EMQX process memory and Erlang VM metrics
2. Decide whether sandbox should be fully offline for the low-traffic month. If yes, prepare a snapshot-and-delete plan for sandbox NAT, ALBs, RDS, Redis, stopped EC2 storage, and DNS dependencies.
3. Prioritise `odoo-production` io1 to gp3 as the cleanest production cost optimisation, subject to maintenance-window approval.
4. Run an ownership pass for unattached EBS volumes and idle EIPs, then batch them into approval-ready cleanup tickets.
5. Keep the production MQTT broker and production DBs online until exact dependency evidence is collected.

## Open Questions

- How many MQTT clients are currently connected to EMQX, and are they real hubs/devices or only backend keep-alives?
- Are any customers expected to use sandbox during the quiet month?
- What recovery target is acceptable if sandbox infrastructure is deleted rather than merely stopped?
- Are the 24 unattached EBS volumes tied to any manual rollback or forensic retention expectation?
- Are the 8 idle Elastic IPs referenced by external parties, old DNS records, or vendor allowlists?
