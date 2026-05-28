# SensorSyn Infrastructure Inventory — April 2026

**Prepared:** 2026-04-20  
**Updated:** 2026-04-22 — UAT, QA, Dev environments fully decommissioned  
**AWS Account:** `747293622182`, primary region `ap-southeast-2`  
**Currency:** AUD (1 USD = 1.58 AUD, April 2026)  
**Cost basis:** AWS figures from AWS Cost Explorer (actual billing). SaaS from prior investigations where available; manual-fill fields marked `[FILL]`.  
**April 2025 note:** Cost Explorer returned no data for April 2025 — account was not materially active pre-production. Year-over-year comparison unavailable; Feb/Mar/Apr 2026 trend is shown instead.  
**XLS export:** Each `[Tab: ...]` heading maps to one Excel worksheet. See [Section 13](#13-xls-export-guide) for paste instructions.

---

## 1. All-Services Cost Summary `[Tab: Overview]`

### 1.1 AWS Monthly Run-Rate (AUD, actual billing)

> **Note:** UAT, QA and Dev environments were decommissioned 2026-04-21/22. April costs include those environments for ~21 days. The "May+ est." column reflects the reduced prod+sandbox-only steady state.

| Service | Feb 2026 | Mar 2026 ¹ | Apr 2026 MTD (22d) | Apr 2026 proj. ² | May+ est. ³ |
|---|---:|---:|---:|---:|---:|
| EC2 — Compute | 2,609 | 2,762 | 1,453 | ~1,981 | ~1,450 |
| EC2 — Other (EBS, data xfer, EIPs) | 2,104 | 2,181 | 1,393 | ~1,899 | ~1,300 |
| RDS | 1,578 | 1,682 | 1,369 | ~1,866 | ~1,340 |
| CloudWatch | 1,633 | 1,832 | 1,010 | ~1,376 | ~950 |
| Tax (GST) | 915 | 1,002 | 644 | ~877 | ~700 |
| ElastiCache | 510 | 568 | 542 | ~738 | ~280 |
| OpenVPN (Marketplace) | 653 | 690 | 283 | ~386 | ~390 |
| VPC (NAT GW, flow logs, traffic) | 429 | 487 | 326 | ~444 | ~200 |
| ELB (load balancers) | 350 | 392 | 280 | ~381 | ~190 |
| Amazon QuickSight | 161 | 161 | 110 | ~150 | ~150 |
| AWS WAF | 99 | 99 | 68 | ~92 | ~90 |
| AWS Security Hub | 77 | 101 | 68 | ~93 | ~90 |
| AWS Config | 86 | 95 | 68 | ~93 | ~90 |
| Amazon Inspector | 101 | 111 | 63 | ~86 | ~80 |
| Amazon S3 | 61 | 61 | 54 | ~73 | ~55 |
| Amazon GuardDuty | 51 | 57 | 33 | ~46 | ~40 |
| AWS Global Accelerator | 27 | 29 | 19 | ~20 ⁴ | — |
| Amazon Macie | 17 | 18 | 14 | ~19 | ~15 |
| Amazon Route 53 | 12 | 12 | 12 | ~16 | ~16 |
| AWS Systems Manager | 13 | 14 | 10 | ~13 | ~12 |
| AWS Key Management Service | 8 | 8 | 5 | ~7 | ~7 |
| AWS Secrets Manager | 8 | 8 | 5 | ~7 | ~7 |
| Amazon Kinesis Firehose | 4 | 4 | 2 | ~3 | ~3 |
| AWS End User Messaging (SMS) | 458 | 707 | 95 | ~129 | ~130 |
| AWS Glue | 2 | 2 | 1 | ~1 | ~1 |
| **Vanta (Marketplace, 6-mo lump)** | — | **9,875** | — | — | — |
| **AWS Total (excl. Vanta & Tax)** | **~10,487** | **~12,041** | **~6,611** | **~9,020** | **~7,585** |
| **AWS Total (incl. Tax, excl. Vanta)** | **~11,964** | **~13,642** | **~7,458** | **~10,613** ⁵ | **~8,285** |

¹ March 2026 Vanta charge (A$9,875 / US$6,250) is a 6-month lump sum. Amortised: ~A$1,645/mo. Next payment expected September 2026.  
² April 2026 full-month projection: linear extrapolation of 22-day actuals (×30/22). Inflated because QA/Dev/UAT ran for ~21 of those days.  
³ May+ estimate: Apr proj. adjusted down for removed QA/Dev/UAT resources. Key drivers: -3 NAT GWs (~A$204), -8 RDS instances (~A$620), -6 ElastiCache nodes (~A$520), -7 ALBs (~A$115), -various EBS volumes. Directional estimate only.  
⁴ Global Accelerator `SmokedevelopmentLBAccelerator` deleted 2026-04-22; no recurrence from May.  
⁵ Apr 2026 total is slightly lower than Feb/Mar due to smaller QA/Dev/UAT footprint (EC2 servers were already terminated before this inventory).

### 1.2 All-Provider Summary

| Provider / Service | Est. A$/mo | Billing method | Data quality |
|---|---:|---|---|
| AWS (base run-rate, prod+sandbox only, May+ est.) | ~8,285 | AWS billing | ✅ Estimated from Apr actuals |
| Vanta | ~1,645 amortised | AWS Marketplace, 6-mo lump | ✅ Confirmed (last payment Mar 2026) |
| New Relic (SaaS subscription) | [FILL] | Direct NR billing | ⚠️ Manual fill |
| MongoDB Atlas | [FILL] | Atlas billing | ⚠️ Manual fill |
| KORE SIM (IoT cellular) | [FILL] | KORE portal | ⚠️ Manual fill |
| Twilio | [FILL] | Twilio billing | ⚠️ Manual fill |
| SendGrid (email) | [FILL] | SendGrid billing | ⚠️ Manual fill |
| Sentry (error monitoring) | [FILL] | Sentry billing | ⚠️ Manual fill |
| LaunchDarkly (feature flags) | [FILL] | LD billing | ⚠️ Manual fill |
| Bitrise (mobile CI/CD) | [FILL] | Bitrise billing | ⚠️ Manual fill |
| Microsoft 365 | [FILL] | Microsoft billing | ⚠️ Manual fill |
| GitHub | [FILL] | GitHub billing | ⚠️ Manual fill |
| Firebase / GCP | [FILL] | GCP billing | ⚠️ Manual fill |
| PropertyMe / PropertyTree | Free API or [FILL] | Integration API | ⚠️ Confirm if paid |
| **TOTAL (AWS + Vanta)** | **~9,930 + [FILL]** | | |

---

## 2. AWS — EC2 Instances `[Tab: EC2]`

On-demand list-price estimates (ap-southeast-2, Linux). Actual billing may be lower if RIs are active. All instances in `ap-southeast-2`. UAT, QA, and Dev environments fully decommissioned 2026-04-21/22.

### 2.1 Production

| Name | Instance ID | Type | vCPU | RAM | State | Private IP | Public IP | Est. A$/mo |
|---|---|---|---|---|---|---|---|---:|
| prod-api-server | i-0fb2cfebc22727abc | t3.medium | 2 | 4 GB | running | 10.0.6.169 | — | ~60 |
| prod-api-server | i-09acc276e638ba099 | t3.medium | 2 | 4 GB | running | 10.0.4.206 | — | ~60 |
| prod-api-server | i-0bcc2dcab7c6dfedf | t3.medium | 2 | 4 GB | running | 10.0.4.248 | — | ~60 |
| prod-api-server | i-0b94a530829f29b4b | t3.medium | 2 | 4 GB | running | 10.0.5.154 | — | ~60 |
| prod-sso-api | i-0d120e1ad6a527ea0 | t3.small | 2 | 2 GB | running | 10.0.4.129 | — | ~32 |
| Emqx-prod | i-0b381a2bfdf65a329 | c5.xlarge | 4 | 8 GB | running | 10.0.2.98 | 13.211.102.112 | ~246 |
| Odoo-production | i-0c5073e95aba77614 | t3.large | 2 | 8 GB | running | 10.0.2.161 | 54.253.158.197 | ~120 |
| Power-BI-Gateway | i-0908eff051972cd8c | t3.large | 2 | 8 GB | **stopped** | 10.0.4.167 | — | ~0 ⁶ |
| Smoke-prod-Keychain-app | i-07b498016eab37ae1 | t3.small | 2 | 2 GB | running | 10.0.4.16 | — | ~32 |
| openvpn-prod | i-00abd001f35108630 | t2.micro | 1 | 1 GB | running | 10.0.3.215 | 13.211.118.57 | ~17 |
| prod-kafka | i-09d176fd791fe7c64 | t3.medium | 2 | 4 GB | running | 10.0.1.199 | 13.237.253.114 | ~60 |
| smoke-prod-api-cron-server | i-0a1cd47cfadedf035 | t2.micro | 1 | 1 GB | running | 10.0.2.167 | 54.153.177.72 | ~17 |
| wordpress-prod | i-0bd9a5e391e812fcc | t3.medium | 2 | 4 GB | running | 10.0.4.29 | — | ~60 |
| wordpress-sensor-insure | i-0dc7e540dc021d6e9 | t2.medium | 2 | 4 GB | running | 10.0.2.37 | 54.252.123.97 | ~71 |
| **Prod subtotal** | | | | | | | | **~895** |

⁶ Power-BI-Gateway is stopped; EBS volume still billing (~A$12/mo). No EIP, so no idle EIP charge.

### 2.2 Sandbox

> ⚠️ **All sandbox EC2 instances are currently stopped.** EIPs remain allocated and billing (~A$6/mo each). RDS and ElastiCache are still running and billing (see §3, §4). Verify if sandbox should be kept warm or further trimmed.

| Name | Instance ID | Type | vCPU | RAM | State | Private IP | Public IP | Est. A$/mo |
|---|---|---|---|---|---|---|---|---:|
| SANDBOX-Kafka-mongo-Smoke | i-0b71daa2680a16a5d | t3.medium | 2 | 4 GB | **stopped** | 10.0.1.210 | 3.105.137.104 | ~6 (EIP) |
| Emqx-sandbox | i-07af1c06915741068 | c5.large | 2 | 4 GB | **stopped** | 10.0.1.127 | 54.66.98.176 | ~6 (EIP) |
| Odoo-sandbox | i-0074106adaa006f2e | t3.medium | 2 | 4 GB | **stopped** | 10.0.2.56 | 3.106.157.12 | ~6 (EIP) |
| openvpn-sandbox | i-0c84b9424bc1ce4dd | t2.micro | 1 | 1 GB | **stopped** | 10.0.3.144 | 54.153.137.92 | ~6 (EIP) |
| smoke-sandbox-api-cron-server | i-008e2acd8997ff315 | t2.micro | 1 | 1 GB | **stopped** | 10.0.1.218 | 54.253.98.34 | ~6 (EIP) |
| **Sandbox subtotal** | | | | | | | | **~30 (EIPs only)** |

### EC2 Summary

| Environment | Running | Stopped | Est. compute A$/mo |
|---|---|---|---:|
| Production | 13 | 1 | ~895 |
| Sandbox | 0 | 5 | ~30 (EIPs) |
| QA | — | — | ✅ decommissioned 2026-04-22 |
| Dev | — | — | ✅ decommissioned 2026-04-22 |
| UAT | — | — | ✅ decommissioned 2026-04-21 |
| **Total** | **13** | **6** | **~925** |

> EC2 billing also includes EBS volumes. The "EC2 Other" line (~A$1,300/mo est.) covers EBS storage/snapshots, data transfer, EIPs, and NAT traffic.

---

## 3. AWS — RDS Databases `[Tab: RDS]`

All in `ap-southeast-2`. UAT, QA, and Dev RDS instances destroyed; final snapshots retained.

| DB Identifier | Engine | Version | Class | Storage | Size | Multi-AZ | Status | Est. A$/mo |
|---|---|---|---|---|---|---|---|---:|
| sensor-prod | MySQL | 8.0.44 | db.t3.medium | gp2 | 20 GB | **Yes** | available | ~179 |
| odoo-production | PostgreSQL | 14.17 | db.t3.medium | io1 | 100 GB | No | available | ~687 ⚠️ |
| sensor-sandbox | MySQL | 8.0.44 | db.t3.medium | gp2 | 20 GB | **Yes** | available | ~179 |
| **Total** | | | | | | | | **~1,045** |

> ⚠️ `odoo-production` uses io1 with 3,000 provisioned IOPS. Storage+IOPS alone ~A$548/mo; peak measured utilisation is 713 IOPS (24%). Migration to gp3 would save ~A$521/mo (previously identified, not yet executed).  
> Actual RDS billing lower due to Reserved Instance discounts.

**Final snapshots from decommissioned environments (retained for 30+ days):**

| Snapshot | Engine | Size | Created |
|---|---|---|---|
| `smokealaramuat-final-20260421` | MySQL | 20 GB | 2026-04-21 |
| `sensor-qa-final-20260421` | MySQL | 50 GB | 2026-04-21 |
| `odoo-qa-final-20260421` | PostgreSQL | 20 GB | 2026-04-21 |
| `dev-db-temp-mysql-8-final-20260421` | MySQL | 50 GB | 2026-04-22 |
| `odoo-dev-final-20260421` | PostgreSQL | 20 GB | 2026-04-22 |

---

## 4. AWS — ElastiCache `[Tab: ElastiCache]`

All clusters: Redis 6.2.6, `ap-southeast-2`. UAT, QA and Dev clusters destroyed.

| Cluster ID | Node Type | Nodes | Status | Est. A$/mo/node |
|---|---|---|---|---:|
| smokeprodapiredis-001 | cache.t3.medium | 1 | available | ~87 |
| smokeprodapiredis-002 | cache.t3.medium | 1 | available | ~87 |
| smokealarmsandbox-001 | cache.t3.medium | 1 | available | ~87 |
| smokealarmsandbox-002 | cache.t3.medium | 1 | available | ~87 |
| **Total** | | **4 nodes** | | **~348** |

| Environment | Nodes | Node type | Est. A$/mo |
|---|---|---|---:|
| Production | 2 | t3.medium | ~174 |
| Sandbox | 2 | t3.medium | ~174 |
| **Total** | **4** | | **~348** |

> Actual ElastiCache billing lower due to Reserved Instance discounts (RI front-loaded on 2026-04-01: A$464).

---

## 5. AWS — S3 Buckets `[Tab: S3]`

~75 buckets in account. Total S3 cost: ~A$73/mo projected Apr 2026.

### 5.1 Production Data

| Bucket | Purpose | Size (known) | Notes |
|---|---|---|---|
| `smokealarmprod-admin` | Angular SPA — all portals | 24 MB / 592 objects | Path-routed via CloudFront |
| `smokeapibucket` | App assets, PDF uploads, exports | 3.4 GB / 31,743 objects | Main application storage |
| `smokealarm-agent` | Legacy portal | [FILL] | Verify if still actively served |
| `smokealarm-trader` | Legacy portal | [FILL] | Verify if still actively served |
| `smokealarm-superadmin` | Legacy portal | [FILL] | Verify if still actively served |

### 5.2 Logs & Audit

| Bucket | Purpose | Environment |
|---|---|---|
| `sensor-global-cloudtrail-logs` | CloudTrail audit (SOC 2 — 365d retention) | Shared |
| `sensor-global-prod-alb-logs` | ALB access logs | Prod |
| `sensor-global-prod-cloudfront-logs` | CloudFront access logs | Prod |
| `sensoralblogs` | ALB log catchall | Prod/shared |
| `smokebucketlogs` | S3 server access logs | Prod |
| `smokesnslogs` | SNS delivery logs | Prod |
| `smoke-system-manager-log` | SSM session/command logs | Shared |
| `smoke-sso-api-lb-logs` | SSO ALB logs | Prod |
| `smoke-wordpress-prod-alb-logs` | WordPress prod ALB logs | Prod |
| `smoke-wordpress-sensorinsure-alb-logs` | SensorInsure ALB logs | Prod |
| `smoke-sandbox-api-lb-logs` | Sandbox ALB logs | Sandbox |
| `smoke-dev-api-lb-logs` | Dev ALB logs | Dev (kept) |
| `smoke-qa-api-lb-logs` | QA ALB logs | QA (kept) |
| `smoke-wordpress-qa-alb-logs` | WordPress QA ALB logs | QA (kept) |
| `smoke-uat-api-alb-logs` | UAT ALB logs | UAT (kept) |
| `smoke-uat-sso-alb-logs` | UAT SSO ALB logs | UAT (kept) |
| `smoke-uat-sso-api-alb-logs` | UAT SSO API ALB logs | UAT (kept) |

### 5.3 Compliance & Security

| Bucket | Purpose |
|---|---|
| `config-bucket-747293622182` | AWS Config snapshots |
| `sensor-securityhub-findings` | Security Hub exports |
| `securityhubcsvmanagerstac-...` | Security Hub CSV exports (CloudFormation-managed) |
| `patch-manager-logs-all-environment` | SSM Patch Manager logs |

### 5.4 Infrastructure / CI-CD

| Bucket | Purpose |
|---|---|
| `sensorsyn-terraform-state-747293622182` | Terraform remote state |
| `codepipeline-ap-southeast-2-...` | CodePipeline artifacts |
| `cf-templates-olkvjcrupnie-ap-southeast-2` | CloudFormation templates |
| `cf-templates-olkvjcrupnie-us-east-1` | CloudFormation templates (us-east-1) |
| `sensor-mysql-snapshot-temp` | Temporary RDS snapshot staging |
| `sensor-newrelic-firehose-17c83850` | New Relic Firehose fallback buffer |
| `aws-athena-query-results-ap-southeast-2-747293622182` | Athena query results |
| `aws-quicksetup-patchpolicy-*` (×15) | AWS Quick Setup Patch Manager (auto-created) |

### 5.5 Non-Production SPA / API Assets (preserved after decommission)

| Bucket | Environment | Purpose |
|---|---|---|
| `smokealarmsandbox-angular` | Sandbox | Angular SPA |
| `smokeapisandboxbucket` | Sandbox | API assets |
| `smokealarmdevelopment-angular` | Dev (kept) | Angular SPA |
| `smokealarmqa-angular` | QA (kept) | Angular SPA |
| `smokealaram-uat-admin` | UAT (kept) | Angular SPA |
| `smokeapidevelopmentbucket` | Dev (kept) | API assets |
| `smokeapiqabucket` | QA (kept) | API assets |
| `sensor-uat-bucket` | UAT (kept) | API assets |

### 5.6 Environment Config (secrets / env vars)

| Bucket | Environment |
|---|---|
| `smokeapienv` | Prod API |
| `ssoapienv` | Prod SSO |
| `smoke-admin-env` | Prod admin |
| `smokeapisandboxenv` | Sandbox API |
| `sandoxssoapienv` | Sandbox SSO |
| `smokeapidevelopmentenv` | Dev (kept) |
| `devssoapienv` | Dev (kept) |
| `smokeapiqaenv` | QA (kept) |
| `qassoapienv` | QA (kept) |
| `sensor-uat-api-env` | UAT (kept) |
| `sensor-uat-sso-env` | UAT (kept) |

---

## 6. AWS — Load Balancers `[Tab: ALBs]`

7 Application Load Balancers, all internet-facing, all `ap-southeast-2`. Base cost ~A$16.43/mo per ALB + variable LCU charges. UAT, QA, and Dev ALBs destroyed.

| Name | Env | DNS Name | Serves |
|---|---|---|---|
| `SmokeAPI-LB` | Prod | SmokeAPI-LB-898969882.ap-southeast-2.elb.amazonaws.com | Prod API |
| `sso-api-lb` | Prod | sso-api-lb-1719844694.ap-southeast-2.elb.amazonaws.com | Prod SSO |
| `keychain-production-ALB` | Prod | keychain-production-ALB-496662693.ap-southeast-2.elb.amazonaws.com | Keychain |
| `wordpress-prod-alb` | Prod | wordpress-prod-alb-1756963168.ap-southeast-2.elb.amazonaws.com | WordPress prod |
| `wordpress-sensorinsure` | Prod | wordpress-sensorinsure-1575846363.ap-southeast-2.elb.amazonaws.com | SensorInsure WP |
| `Smoke-Sandbox-Api-LB` | Sandbox | Smoke-Sandbox-Api-LB-621355096.ap-southeast-2.elb.amazonaws.com | Sandbox API |
| `sandbox-sso-api` | Sandbox | sandbox-sso-api-2060882049.ap-southeast-2.elb.amazonaws.com | Sandbox SSO |
| **Est. total base cost** | | | **~A$115/mo + LCU** |

---

## 7. AWS — Networking `[Tab: Networking]`

### 7.1 NAT Gateways (2 active)

| Gateway ID | VPC | Environment | State | Public IP | Est. A$/mo |
|---|---|---|---|---|---:|
| nat-0419aece84205b4f5 | vpc-03ec09702866ca7b1 | Prod | available | 13.210.248.111 | ~68 + traffic |
| nat-0f00d13d55f11c828 | vpc-0c4eb38fb674b072b | Sandbox | available | 54.253.156.228 | ~68 + traffic |
| **Total NAT GW base** | | | | | **~136/mo** |

### 7.2 Elastic IPs

| Status | Count | Notes |
|---|---|---:|
| In use — running EC2 instances | 8 | Emqx-prod, openvpn-prod, prod-kafka, cron-server, wordpress-sensorinsure, Odoo-production × 2 prod NAT GWs |
| In use — stopped EC2 (EIP billed) | 5 | All sandbox instances (stopped); ~A$6/mo each |
| **Idle / orphaned** | **~8 confirmed** | QA/Dev/UAT-named EIPs not yet released; additional unnamed EIPs need audit |

**Confirmed idle EIPs (released-candidate):**

| Public IP | Name | Was used by |
|---|---|---|
| 13.210.154.83 | Smoke-qa-NAT-Gateway | QA NAT GW (deleted) |
| 13.236.215.14 | smoke-qa-cron-eip | QA cron server (deleted) |
| 13.54.11.168 | Hive-MQ-QA | QA EMQX (deleted) |
| 13.55.2.104 | odoo-qa | QA Odoo (deleted) |
| 3.105.96.66 | prod-mongodb-node-2 | MongoDB node (terminated) |
| 52.64.196.201 | prod-mongodb-node-3 | MongoDB node (terminated) |
| 54.153.181.223 | Smoke-development-NAT-Gateway | Dev NAT GW (deleted) |
| 54.252.83.199 | openvpn-qa | QA OpenVPN (deleted) |

> ⚠️ **At minimum 8 idle EIPs confirmed (~A$46/mo).** A full EIP audit is recommended — the previous inventory identified 58 idle EIPs. Many may still be unattached post-decommission.

---

## 8. AWS — CDN & DNS `[Tab: CDN-DNS]`

### 8.1 CloudFront Distributions (11)

> ⚠️ Dev, QA, and UAT CloudFront distributions are still deployed pointing to preserved S3 buckets. They continue to serve CDN URLs but environments are gone. Consider disabling or deleting UAT/QA/Dev distributions to reduce confusion; cost is negligible (near-zero traffic).

| Distribution ID | CloudFront Domain | Origin (S3 bucket) | Environment |
|---|---|---|---|
| E1UDNFPYDRUZOD | d2obn6q80a5yi1.cloudfront.net | smokealarmprod-admin | **Prod** (all portals) |
| E1US6KVM4RUTH5 | dxu6ibcxyrwaw.cloudfront.net | smokeapibucket | **Prod** (app assets) |
| EOKA8AFVFKKIR | d3d3d51xr64xiy.cloudfront.net | smokealarmsandbox-angular | Sandbox portals |
| E17GFUEXQFZ3KN | d34owv7u0bh3il.cloudfront.net | smokeapisandboxbucket | Sandbox API assets |
| EGCINYRLUW17W | d3v5gqx3r8jm6c.cloudfront.net | smokealarmqa-angular | QA portals (kept) |
| EURY1GYY9JQOQ | do35m176tv0kl.cloudfront.net | smokeapiqabucket | QA API assets (kept) |
| E20OW2MO42EN3D | dwxmdpwkhlndt.cloudfront.net | smokealarmdevelopment-angular | Dev portals (kept) |
| E2WZRVQ69B78C8 | d1ix8fpeaigsnu.cloudfront.net | smokeapidevelopmentbucket | Dev API assets (kept) |
| E3K8XZ4730CNBG | din2iaf8x1far.cloudfront.net | dev-keychain | Dev keychain (kept) |
| E3NLD61VNLP2FE | d2us7idlltmlz4.cloudfront.net | sensor-uat-bucket | UAT API assets (kept) |
| E3ASENEW8EEQ82 | dnaed6u7qv2m4.cloudfront.net | smokealaram-uat-admin | UAT portals (kept) |

### 8.2 Route 53 Hosted Zones (14)

| Domain | Records | Notes |
|---|---|---|
| sensorglobal.com | 119 | Primary — API, SSO, portals, MQTT, Kafka, Odoo |
| sensorinsure.com | 7 | SensorInsure product |
| sensorinsure.com.au | 7 | SensorInsure AU |
| sensorglobal.com.au | 5 | AU variant |
| sensorglobal.net | 4 | Net variant |
| sensorglobal.co.uk | 2 | Brand protection |
| sensorglobal.co.nz | 2 | Brand protection |
| sensorglobal.au | 2 | Brand protection |
| sensoriq.com.au | 2 | Brand protection |
| sensoriq.co.uk | 2 | Brand protection |
| sensoriq.au | 2 | Brand protection |
| sensoriq.net | 2 | Brand protection |
| sensorsmartinsurance.com | 2 | Brand protection |
| sensorsmartinsurance.com.au | 2 | Brand protection |

Route 53 cost: ~A$16/mo (14 zones × US$0.50 + query volume).

---

## 9. AWS — Lambda Functions `[Tab: Lambda]`

31 functions in `ap-southeast-2`. Total Lambda cost is negligible (~$2–5/mo). All are short-duration, event-driven.

| Function Name | Runtime | Memory | Timeout | Purpose | Status |
|---|---|---|---|---|---|
| Smoke_prod_api_golden_ami_function | python3.10 | 128 MB | 10s | Prod API golden AMI refresh | ✅ Active |
| Smoke_prod_sso_api_golden_ami_function | python3.10 | 128 MB | 10s | Prod SSO golden AMI refresh | ✅ Active |
| Smoke_sandbox_api_golden_ami_function | python3.10 | 128 MB | 10s | Sandbox API golden AMI | ✅ Active |
| Smoke_sandbox_sso_api_golden_ami_function | python3.10 | 128 MB | 10s | Sandbox SSO golden AMI | ✅ Active |
| createfilezipprod | nodejs16.x ⚠️ | 512 MB | 60s | File zip — prod | ✅ Active |
| createfilezipsandbox | nodejs16.x ⚠️ | 512 MB | 30s | File zip — sandbox | ✅ Active |
| convert-json-to-csv | python3.13 | 512 MB | 603s | Data conversion | ✅ Active |
| prod-api-cloudwatch-alarm | python3.10 | 128 MB | 3s | Prod CW alarm handler | ✅ Active |
| snstoteams | python3.9 ⚠️ | 128 MB | 3s | SNS → Microsoft Teams | ✅ Active |
| Smoke_qa_api_golden_ami_function | python3.10 | 128 MB | 10s | QA API golden AMI | ⚠️ Orphaned (QA gone) |
| Smoke_qa_sso_api_golden_ami | python3.10 | 128 MB | 10s | QA SSO golden AMI | ⚠️ Orphaned (QA gone) |
| Smoke_dev_sso_api_golden_ami | python3.10 | 128 MB | 10s | Dev SSO golden AMI | ⚠️ Orphaned (Dev gone) |
| createfilezipqa | nodejs16.x ⚠️ | 2,048 MB | 300s | File zip — QA | ⚠️ Orphaned (QA gone) |
| createfilezipdev | nodejs16.x ⚠️ | 512 MB | 30s | File zip — dev | ⚠️ Orphaned (Dev gone) |
| createfilezipuat | nodejs16.x ⚠️ | 128 MB | 3s | File zip — UAT | ⚠️ Orphaned (UAT gone) |
| dev-api-cloudwatch-alarms | python3.10 | 128 MB | 3s | Dev API CW alarms | ⚠️ Orphaned (Dev gone) |
| dev-sso-cloudwatch-alarms | python3.10 | 128 MB | 3s | Dev SSO CW alarms | ⚠️ Orphaned (Dev gone) |
| qa-api-cloudwatch-alarms | python3.10 | 128 MB | 3s | QA API CW alarms | ⚠️ Orphaned (QA gone) |
| qa-sso-cpu-usage-cloudwatch-alarms | python3.10 | 128 MB | 3s | QA SSO CW alarms | ⚠️ Orphaned (QA gone) |
| uat-api-cpu-usage-cloudwatch-alarms | python3.10 | 128 MB | 3s | UAT API CW alarms | ⚠️ Orphaned (UAT gone) |
| uat-sso-cpu-usage-cloudwatch-alarm | python3.10 | 128 MB | 3s | UAT SSO CW alarms | ⚠️ Orphaned (UAT gone) |
| baseline-overrides-* (×5) | python3.11 | 128 MB | 300s | AWS Quick Setup | System-managed |
| delete-name-tags-* (×5) | python3.11 | 128 MB | 900s | AWS Quick Setup | System-managed |

> ⚠️ All `createfilezip*` functions use `nodejs16.x` (EOL). Upgrade active ones to Node.js 22; delete orphaned ones.  
> ⚠️ `snstoteams` uses `python3.9` (EOL). Upgrade to python3.12 or 3.13.  
> ⚠️ 9 Lambda functions now orphaned (QA/Dev/UAT gone) — safe to delete.

---

## 10. GCP / Firebase `[Tab: GCP-Firebase]`

No GCP CLI access. Fill manually from [GCP Billing Console](https://console.cloud.google.com/billing).

| Resource | GCP Project | Service | Purpose | A$/mo | Notes |
|---|---|---|---|---|---|
| Firebase project | sensor-production | Firebase Cloud Messaging (FCM) | Mobile/web push notifications | [FILL] | Referenced in `sensor-prod` AWS secret |
| reCAPTCHA Enterprise | [FILL] | reCAPTCHA Enterprise | Login form bot protection | [FILL] | `RECAPTCHA_PROJECTID` in sensor-prod secret |
| Google Maps API | [FILL] | Maps JS / Geocoding API | Property map views | [FILL] | `GOOGLE_MAPS_API_KEY` in sensor-prod secret |
| Corpsure integration | [FILL] | Cloud Functions | Insurance partner endpoint | [FILL] | `CORPSURE_*` credentials in sensor-prod secret |
| **GCP Total** | | | | **[FILL A$]** | |

---

## 11. GitHub `[Tab: GitHub]`

Fill manually from [GitHub Billing Settings](https://github.com/organizations/SensorGlobal/settings/billing).

| Item | Value |
|---|---|
| Organisation | SensorGlobal |
| Plan | [FILL: Team / Enterprise / Free] |
| Seats (billed) | [FILL] |
| Seat cost (/mo) | [FILL] |
| GitHub Actions minutes used (last 30d) | [FILL] |
| Actions included in plan | [FILL] |
| Actions overage cost | [FILL] |
| Package / LFS storage | [FILL] |
| **Total monthly cost** | **[FILL]** |

Active repositories (from codebase investigation):
- `sensor-alarm-backend` — main Node.js/Express backend API
- `sensor-angular` — Angular portals (admin, agent, trader, superadmin, keychain, consumer)
- `sso-provider` — OIDC/SSO service
- `mqtt-prototype` — .NET IoT hub/controller simulator
- `odoo-*` — Odoo customisations and supporting material

CI/CD uses GitHub Actions with OIDC authentication to AWS for Terraform operations.

---

## 12. SaaS & Third-Party Services `[Tab: SaaS]`

| Service | Purpose | Billing cadence | Est. A$/mo | Source | Notes |
|---|---|---|---|---|---|
| **New Relic** | APM, metrics, distributed tracing | Direct NR subscription | [FILL NR sub] + ~A$1,376 CW | NR billing + AWS CE | CloudWatch cost partly driven by NR metric stream. Check NR account billing separately. |
| **Vanta** | SOC 2 security compliance automation | AWS Marketplace, 6-mo lump | ~A$1,645 amortised | AWS CE | A$9,875 / US$6,250 per 6 months. Last payment Mar 2026; next expected Sep 2026. |
| **OpenVPN** | Developer remote access VPN | AWS Marketplace, per-hr | ~A$390 | AWS CE | EC2 instance `i-00abd001f35108630` (t2.micro). |
| **MongoDB Atlas** | Document DB (SSO + app data) | Atlas subscription | [FILL] | atlas.mongodb.com | Cluster `sensor-prod.vr48f.mongodb.net`. Check Atlas org billing. |
| **KORE SIM** | IoT hub cellular backhaul (LTE Cat-M) | KORE portal, per-SIM/mo | [FILL] | kore.com portal | All field sensor hubs connect via KORE SIM. |
| **Twilio** | SMS (supplementary / fallback) | Twilio billing | [FILL] | Twilio console | AWS End User Messaging (~A$129/mo) handles primary SMS. Twilio may be fallback or legacy. |
| **SendGrid** | Transactional + compliance email | SendGrid billing | [FILL] | SendGrid console | 2 API keys in sensor-prod secret. |
| **Sentry** | Error monitoring | Sentry billing | [FILL] | sentry.io | `SENTRY_DSN` in sensor-prod secret. |
| **LaunchDarkly** | Feature flags | LD billing | [FILL] | launchdarkly.com | SDK key in sensor-prod secret. |
| **Bitrise** | Mobile app CI/CD | Bitrise billing | [FILL] | bitrise.io | `bitrise` IAM user exists. |
| **Microsoft 365** | Corporate email + Microsoft services | Microsoft billing | [FILL] | Microsoft Admin Centre | MX → `protection.outlook.com`. |
| **QuickSight** | BI / reporting dashboards | AWS billing | ~A$150 | AWS CE | Included in AWS cost table. |
| **PropertyMe** | Property management integration | API (confirm if paid) | [FILL] | PropertyMe portal | API key in sensor-prod secret. |
| **PropertyTree** | Property management integration | API (confirm if paid) | [FILL] | PropertyTree portal | API key in sensor-prod secret. |
| **Console Cloud** | Property management integration | API (confirm if paid) | [FILL] | Console portal | API key in sensor-prod secret. |
| **NEXU** | [FILL — purpose unknown] | [FILL] | [FILL] | NEXU portal | `NEXU_*` credentials in sensor-prod secret. |
| **Corpsure** | Insurance partner integration | [FILL] | [FILL] | Corpsure portal | Cloud Functions + credentials in sensor-prod secret. |

---

## 13. XLS Export Guide `[Tab: Export-Guide]`

Each Markdown table can be pasted directly into Excel or Google Sheets.

| Section | Excel tab name | Content to copy |
|---|---|---|
| §1 All-Services Cost Summary | **Overview** | Tables 1.1, 1.2 |
| §2 EC2 Instances | **EC2** | All env tables + summary |
| §3 RDS | **RDS** | Main table + snapshots table |
| §4 ElastiCache | **ElastiCache** | Full table |
| §5 S3 | **S3** | All 6 sub-tables |
| §6 Load Balancers | **ALBs** | Full table |
| §7 Networking | **Networking** | NAT GW table + EIP table |
| §8 CDN & DNS | **CDN-DNS** | CloudFront table + Route 53 table |
| §9 Lambda | **Lambda** | Full table |
| §10 GCP / Firebase | **GCP-Firebase** | Fill-in table |
| §11 GitHub | **GitHub** | Key-value block |
| §12 SaaS | **SaaS** | Full table |

**Paste tips:**
- Google Sheets: paste Markdown table → Data → Split text to columns → pipe `|` delimiter
- Excel: paste → Text to Columns → Delimited → `|`
- Alternatively use [tableconvert.com](https://tableconvert.com/markdown-to-excel)

---

## Key Findings & Action Items

| # | Finding | Est. monthly saving (A$) | Effort | Status |
|---|---|---:|---|---|
| 1 | UAT, QA, Dev decommissioned | ~A$2,800–3,500 ⁷ | ✅ Done | Completed 2026-04-22 |
| 2 | `odoo-production` RDS io1 → gp3 migration | ~A$521 | Medium | Identified, not yet executed |
| 3 | 8+ idle Elastic IPs (QA/Dev/UAT-named, post-teardown) | ~A$46+ | Low | Ready to release |
| 4 | 9 orphaned Lambda functions (QA/Dev/UAT) | Negligible | Low | Safe to delete |
| 5 | Sandbox all-stopped: RDS + ElastiCache still billing (~A$527/mo) with no EC2 | ~A$527 | Low-Medium | Review sandbox keep vs. teardown |
| 6 | CloudWatch costs (~A$1,376/mo) — New Relic metric stream volume | TBD | Medium | Investigate metric cardinality |
| 7 | `createfilezip*` Lambda on EOL nodejs16.x (5 functions) | Security only | Low | Upgrade active 2; delete orphaned 3 |
| 8 | `snstoteams` Lambda on EOL python3.9 | Security only | Low | Upgrade to 3.12+ |
| 9 | QA/Dev/UAT CloudFront distributions still deployed (S3 kept) | Negligible | Low | Disable or leave |
| 10 | Full EIP audit — pre-decommission count was 58 idle; recount needed | TBD | Low | Run `aws ec2 describe-addresses` audit |

⁷ Estimated decommissioning saving: RDS ~A$620 + ElastiCache ~A$520 + ALBs ~A$115 + NAT GWs ~A$204 + EBS/EIPs ~A$150 + EC2 (was already gone) = ~A$1,600/mo. Full impact depends on what QA/Dev EC2 was previously billing.

---

*Data sources: AWS Cost Explorer, `aws ec2/rds/elasticache/elbv2/lambda/s3api/cloudfront/route53` describe/list commands — all queried 2026-04-22.*  
*SaaS costs from prior investigation docs where available; `[FILL]` fields require manual entry from vendor billing portals.*
