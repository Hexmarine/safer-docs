# Runbook 10 - Sandbox Fixed-Cost Cleanup

**Backlog item:** `OPT-001 Sandbox fixed-cost cleanup`  
**Created:** 2026-04-29  
**Status:** Stage 1, stage 2A, and stage 2B applied externally and verified  
**Approval reference:** User replied `proceed` on 2026-04-29; default posture is `Archive then delete`  
**Applies to:** sandbox resources in AWS account `747293622182`, region `ap-southeast-2`  
**Primary evidence:** `docs/investigations/2026-04-29-low-traffic-mqtt-and-optimisation.md`

## Decision Needed

Choose the recovery posture before any remote change:

| Option | Meaning | Recommended when |
|---|---|---|
| Pause | Stop only resources that can be stopped; keep data/network shell mostly intact | Sandbox may be needed soon |
| Archive then delete | Take final snapshots/backups, then delete fixed-cost resources | Sandbox is not needed next month but may need rebuild |
| Retire | Delete sandbox resources after final confirmation; keep only durable records/docs | Sandbox is no longer needed |

Recommended default for the expected low-traffic month: **Archive then delete** for fixed-cost resources, but keep shared/global resources untouched.

## Read-Only Baseline

Identity check:

| Item | Value |
|---|---|
| AWS account | `747293622182` |
| Region | `ap-southeast-2` |
| Caller ARN observed | `arn:aws:iam::747293622182:user/peresada1@gmail.com` |

Terraform check:

| Check | Result |
|---|---|
| `terraform -chdir=code/infra/terraform/environments/sandbox validate -no-color` | Valid |
| `terraform -chdir=code/infra/terraform/environments/sandbox plan -lock=false -input=false -no-color -detailed-exitcode` | No changes |

Note: AWS CLI currently lists only the two production ASGs. Terraform state still contains historical sandbox ASG addresses, but sandbox launch-template/ASG app capacity is not the current cost driver. Treat Terraform/state reconciliation as a separate guardrail before any Terraform-driven cleanup.

## Sandbox Resource Inventory

### Network

| Resource | Current state | Notes |
|---|---|---|
| VPC `vpc-0c4eb38fb674b072b` | Present | `Smoke-Sandbox-VPC` |
| NAT Gateway `nat-0f00d13d55f11c828` | `available` | Public IP `54.253.156.228`; 0 bytes and 0 packets over the 2026-04-22 to 2026-04-29 CloudWatch window |
| EIP `eipalloc-0121eef619a926133` | Associated to NAT ENI `eni-0cbb55e91758913fe` | Name tag `Smoke-Sandbox-NAT-GAteway` |
| Private route table `rtb-03fb390cce62724fc` | Active default route to NAT | `0.0.0.0/0 -> nat-0f00d13d55f11c828` |

### Load Balancers

| ALB | State | DNS | Listener shape | Targets |
|---|---|---|---|---|
| `Smoke-Sandbox-Api-LB` | `active` | `Smoke-Sandbox-Api-LB-621355096.ap-southeast-2.elb.amazonaws.com` | HTTP 80 redirects to HTTPS 443; HTTPS forwards to `Smoke-Sandbox-API-TG` | No registered targets |
| `sandbox-sso-api` | `active` | `sandbox-sso-api-2060882049.ap-southeast-2.elb.amazonaws.com` | HTTP 80 redirects to HTTPS 443; HTTPS forwards to `sandbox-sso-api-tg` | No registered targets |

Both sandbox ALBs had 0 requests over the 2026-04-22 to 2026-04-29 CloudWatch window.

### DNS And Frontend

| Record | Current value | Cleanup implication |
|---|---|---|
| `api-sandbox.sensorglobal.com` | CNAME to `Smoke-Sandbox-Api-LB-621355096.ap-southeast-2.elb.amazonaws.com` | Must delete, park, or retarget before deleting ALB |
| `auth-sandbox.sensorglobal.com` | CNAME to `sandbox-sso-api-2060882049.ap-southeast-2.elb.amazonaws.com` | Must delete, park, or retarget before deleting ALB |
| `mqtt-sandbox.sensorglobal.com` | A record `54.66.98.176` | Points to stopped EMQX sandbox instance |
| `crm-sandbox.sensorglobal.com` | A record `3.106.157.12` | Points to stopped Odoo sandbox instance |
| `kafkasandbox.sensorglobal.com` | A record `3.105.137.104` | Points to stopped Kafka/Mongo sandbox instance |
| `admin/agent/trader/activation/reschedule/owner-sandbox.sensorglobal.com` | CNAME to CloudFront `d3d3d51xr64xiy.cloudfront.net` | Frontend distribution remains enabled |
| `keychain-sandbox.sensorglobal.com` | CNAME to shared production keychain ALB | Do not change as part of sandbox cleanup without a separate keychain decision |

CloudFront distribution `EOKA8AFVFKKIR` is enabled for the sandbox web frontends and uses S3 origin `smokealarmsandbox-angular.s3.ap-southeast-2.amazonaws.com`.

### Data Stores

| Resource | Current state | Recovery notes |
|---|---|---|
| RDS `sensor-sandbox` | `available`, `db.t3.medium`, MySQL `8.0.44`, Multi-AZ, gp2, 20 GB | 7-day automated backups; latest restorable time observed `2026-04-28T20:38:44Z`; 0 DB connections over the measured week |
| ElastiCache `smokealarmsandbox` | `available`, Redis, 2 x `cache.t3.medium` | No ElastiCache snapshots returned for the replication group; memory averaged about 0.5% |

### Stopped EC2 And Attached Storage

| Instance | State | Type | Public IP / EIP | Attached volume |
|---|---|---|---|---|
| `i-0074106adaa006f2e` `Odoo-sandbox` | stopped | `t3.medium` | `3.106.157.12` / `eipalloc-04b3345e18629d6f4` | `vol-0a89d230e0f94588b`, 60 GB gp2 |
| `i-0b71daa2680a16a5d` `SANDBOX-Kafka-mongo-Smoke` | stopped | `t3.medium` | `3.105.137.104` / `eipalloc-05cdec5ea2b6d6c5b` | `vol-0d0627791dfcd6d78`, 50 GB gp2 |
| `i-008e2acd8997ff315` `smoke-sandbox-api-cron-server` | stopped | `t2.micro` | `54.253.98.34` / `eipalloc-028c41ffb5241e124` | `vol-0a4f037bb9b336f0d`, 8 GB gp2 |
| `i-07af1c06915741068` `Emqx-sandbox` | stopped | `c5.large` | `54.66.98.176` / `eipalloc-0f3b645530195c938` | `vol-01b3c948846ae36f8`, 30 GB gp2 |
| `i-0c84b9424bc1ce4dd` `openvpn-sandbox` | stopped | `t2.micro` | `54.153.137.92` / `eipalloc-01577248aa180619e` | `vol-0154f6eb5a20a6e99`, 20 GB gp2 |

These instances are not incurring EC2 compute while stopped, but their EBS volumes and associated EIPs still have cost impact.

### S3

Sandbox-named buckets discovered:

- `smoke-sandbox-api-lb-logs`
- `smokealarmsandbox-angular`
- `smokeapisandboxbucket`
- `smokeapisandboxenv`

Do not delete S3 buckets in the first cleanup pass unless the team explicitly chooses `Retire` and confirms archive/export requirements.

## Recommended Execution Sequence

All steps below are remote infrastructure mutations and require explicit approval. Codex must not run them under the repository guardrails.

### Phase 0 - Approval And Freeze

1. Confirm chosen recovery posture: `Pause`, `Archive then delete`, or `Retire`.
2. Confirm date range for sandbox unavailability.
3. Confirm no customer demo, test hub, mobile app validation, vendor handover, or migration dry run depends on sandbox.
4. Confirm whether DNS names should be deleted, parked, or left dangling temporarily after backend removal.

### Phase 1 - Final Recovery Points

1. Create final manual snapshot for `sensor-sandbox`.
2. Create final ElastiCache snapshot for `smokealarmsandbox` if Redis is not confirmed transient.
3. Create EBS snapshots for the five stopped sandbox EC2 volumes if instance-level recovery is required.
4. Export or archive any required S3 content before deleting buckets or CloudFront later.

### Phase 2 - Highest Fixed-Cost Removals

1. Delete or stop `sensor-sandbox`.
   - For one month only, stopping can work but must be repeated before AWS auto-restarts it.
   - For durable savings, delete after final snapshot.
2. Delete `smokealarmsandbox` if Redis is confirmed transient or final snapshot is accepted.
3. Delete the sandbox NAT Gateway after private-subnet dependency confirmation.
4. Remove or update the private route-table default route that currently points to the sandbox NAT Gateway.

### Phase 3 - Public Endpoint Cleanup

1. Decide DNS outcome for `api-sandbox.sensorglobal.com` and `auth-sandbox.sensorglobal.com`.
2. Delete sandbox ALB listeners, target groups, and ALBs after DNS is removed or parked.
3. Confirm no CloudWatch alarms, dashboards, or external monitors still expect the ALBs.

### Phase 4 - Stopped Instance Cleanup

1. For each stopped sandbox EC2 instance, confirm whether a snapshot is sufficient for recovery.
2. Terminate approved instances.
3. Delete approved attached EBS volumes after snapshot completion.
4. Release approved EIPs after DNS and vendor allowlist checks.

### Phase 5 - Optional Frontend And Bucket Cleanup

1. Disable/delete CloudFront distribution `EOKA8AFVFKKIR` only if sandbox web frontends are retired.
2. Archive and delete sandbox S3 buckets only if the team chooses full retirement.
3. Keep `keychain-sandbox.sensorglobal.com` untouched unless there is a separate keychain plan, because it points at a shared production keychain ALB.

## Terraform-First Execution Path

Most OPT-001 resources are already managed in `code/infra/terraform/environments/sandbox`, so the preferred execution path is Terraform-first rather than one-off AWS CLI deletion.

Resources currently managed by sandbox Terraform and in OPT-001 scope:

- `aws_db_instance.sensor_sandbox`
- `aws_elasticache_replication_group.smokealarmsandbox`
- `aws_nat_gateway.sandbox_nat_gw`
- `aws_eip.sandbox_nat_eip`
- `aws_lb.sandbox_api_lb`
- `aws_lb.sandbox_sso_lb`
- `aws_lb_target_group.sandbox_api_tg`
- `aws_lb_target_group.sandbox_sso_tg`
- sandbox ALB listeners
- stopped sandbox EC2 instances
- sandbox EC2 EIPs and EIP associations

Resources not currently covered by sandbox Terraform:

- Route53 records such as `api-sandbox.sensorglobal.com`, `auth-sandbox.sensorglobal.com`, `mqtt-sandbox.sensorglobal.com`, `crm-sandbox.sensorglobal.com`, and `kafkasandbox.sensorglobal.com`
- CloudFront distribution `EOKA8AFVFKKIR`
- shared keychain endpoint `keychain-sandbox.sensorglobal.com`

### Terraform stage 1 - prepare for decommission

The local Terraform diff now prepares the selected fixed-cost resources for a later destroy by:

- setting `aws_db_instance.sensor_sandbox.deletion_protection = false`
- setting `aws_db_instance.sensor_sandbox.skip_final_snapshot = false`
- setting `aws_db_instance.sensor_sandbox.delete_automated_backups = false`
- setting `aws_db_instance.sensor_sandbox.final_snapshot_identifier = "sensor-sandbox-final-${var.opt001_final_snapshot_suffix}"`
- setting `aws_elasticache_replication_group.smokealarmsandbox.final_snapshot_identifier = "smokealarmsandbox-final-${var.opt001_final_snapshot_suffix}"`
- setting `aws_lb.sandbox_api_lb.enable_deletion_protection = false`
- setting `aws_lb.sandbox_sso_lb.enable_deletion_protection = false`

Validation result:

```text
terraform -chdir=code/infra/terraform/environments/sandbox validate -no-color
Success! The configuration is valid.
```

Plan result:

```text
Plan: 0 to add, 4 to change, 0 to destroy.
```

The four in-place changes are:

1. `aws_db_instance.sensor_sandbox`
2. `aws_elasticache_replication_group.smokealarmsandbox`
3. `aws_lb.sandbox_api_lb`
4. `aws_lb.sandbox_sso_lb`

This stage must be applied before removing the resource blocks; otherwise Terraform would try to delete resources using the old state values, including RDS `skip_final_snapshot = true` and ALB deletion protection still enabled.

Stage 1 was externally applied and verified on 2026-04-29. Sandbox Terraform now plans cleanly, RDS `sensor-sandbox` has deletion protection disabled, and both sandbox ALBs have deletion protection disabled.

### Terraform stage 2A - prepare stopped EC2 for decommission

The local Terraform diff now prepares the stopped sandbox EC2 instances for later Terraform destroy by:

- setting `aws_instance.kafka_mongo_sandbox.disable_api_termination = false`
- setting `aws_instance.emqx_sandbox.disable_api_termination = false`
- setting `aws_instance.odoo_sandbox.disable_api_termination = false`
- setting `delete_on_termination = true` for the root volumes on `cron_sandbox`, `kafka_mongo_sandbox`, and `odoo_sandbox`

Validation result:

```text
terraform -chdir=code/infra/terraform/environments/sandbox validate -no-color
Success! The configuration is valid.
```

Plan result:

```text
Plan: 0 to add, 4 to change, 0 to destroy.
```

The four in-place changes are:

1. `aws_instance.cron_sandbox`
2. `aws_instance.emqx_sandbox`
3. `aws_instance.kafka_mongo_sandbox`
4. `aws_instance.odoo_sandbox`

This stage must be applied before removing the EC2 resource blocks; otherwise Terraform may fail on API termination protection or leave root EBS volumes orphaned.

Stage 2A was externally applied and verified on 2026-04-29. Sandbox Terraform planned cleanly after apply, and read-only EC2 checks showed API termination protection disabled on the previously protected stopped instances.

### Terraform stage 2B - decommission selected resources

After stage 2A is applied externally and verified, the local Terraform diff removes the selected fixed-cost resources from active `.tf` files while keeping the wider sandbox shell in place.

Minimum stage-2 destroy scope:

- RDS: `aws_db_instance.sensor_sandbox`
- ElastiCache: `aws_elasticache_replication_group.smokealarmsandbox`
- NAT: `aws_nat_gateway.sandbox_nat_gw`, `aws_eip.sandbox_nat_eip`, and the NAT default route from `aws_route_table.sandbox_private_rt`
- ALB: sandbox API/SSO ALBs, listeners, and target groups
- stopped EC2: sandbox Odoo, Kafka/Mongo, cron, EMQX, OpenVPN instances
- EC2 public IPs: sandbox Odoo, Kafka/Mongo, cron, EMQX, OpenVPN EIPs and EIP associations

Keep out of stage 2 unless there is a separate retirement decision:

- sandbox S3 buckets
- CloudFront distribution and frontend DNS
- shared keychain endpoint
- VPC, subnets, internet gateway, security groups, Secrets Manager secrets, log groups, and alarms

Stage-2 apply should be preceded by DNS cleanup/import decisions. If Route53 remains unmanaged, delete or park sandbox records outside Terraform before deleting the ALBs and EIPs.

Validation result:

```text
terraform -chdir=code/infra/terraform/environments/sandbox validate -no-color
Success! The configuration is valid.
```

Initial stage-2B plan result:

```text
Plan: 0 to add, 1 to change, 27 to destroy.
```

The first external apply partially completed and reported stale ELB listener errors plus a NAT EIP association error. Read-only checks showed that the main decommission had succeeded and only the unassociated NAT EIP remained in Terraform.

Finish plan result:

```text
Plan: 0 to add, 0 to change, 1 to destroy.
```

Stage 2B was externally completed and verified on 2026-04-29. The final read-only Terraform plan showed:

```text
No changes. Your infrastructure matches the configuration.
```

Read-only AWS cross-checks showed:

- `sensor-sandbox` is not found
- `smokealarmsandbox` is not found
- NAT Gateway `nat-0f00d13d55f11c828` is `deleted`
- NAT EIP `eipalloc-0121eef619a926133` is not found
- the five stopped sandbox EC2 instances are terminated
- the five sandbox instance EIPs are released
- sandbox API/SSO ALBs are not found

Stage 2B intentionally keeps the sandbox VPC, subnets, internet gateway, security groups, subnet groups, launch templates, S3 buckets, Secrets Manager secrets, log groups, alarms, CloudFront distribution, and shared keychain endpoint out of the first decommission apply.

Route53 sandbox records are still outside this Terraform stage. Treat any sandbox records pointing at deleted ALBs or released EIPs as DNS cleanup follow-up rather than part of the fixed-cost decommission verification.

### Rough stage 2B savings estimate

Estimated monthly gross saving, using 730 hours/month and current AWS public/list rates from the Pricing API where available: about **USD 445/month**, or about **AUD 630/month** using the same rough FX rate used in the April cost investigation.

| Stage 2B area | Approx. monthly saving |
|---|---:|
| RDS `sensor-sandbox` `db.t3.medium` Multi-AZ plus 20 GB gp2 | `USD 155` / `AUD 219` |
| ElastiCache `smokealarmsandbox`, 2 x `cache.t3.medium` | `USD 146` / `AUD 206` |
| NAT Gateway plus NAT public IPv4 | `USD 47` / `AUD 66` |
| Two sandbox ALBs plus estimated ALB public IPv4 addresses | `USD 59` / `AUD 83` |
| Stopped EC2 root EBS volumes, 168 GB gp2, plus five EIPs | `USD 38` / `AUD 54` |
| **Gross recurring subtotal** | **`USD 445` / `AUD 630`** |

Treat this as an order-of-magnitude estimate. Actual bill impact may differ because of reservations, discounts, taxes, partial-month timing, public IPv4 accounting for load balancers, and retained final snapshots/backups. Final RDS, Redis, and EBS snapshots will create a small residual storage cost after deletion.

## Verification After External Changes

Run read-only checks after each externally applied phase:

```bash
aws ec2 describe-nat-gateways --region ap-southeast-2
aws elbv2 describe-load-balancers --region ap-southeast-2
aws rds describe-db-instances --region ap-southeast-2
aws elasticache describe-replication-groups --region ap-southeast-2
aws ec2 describe-instances --region ap-southeast-2
aws ec2 describe-volumes --region ap-southeast-2
aws ec2 describe-addresses --region ap-southeast-2
terraform -chdir=code/infra/terraform/environments/sandbox plan -lock=false -input=false -no-color -detailed-exitcode
```

Expected end state depends on the chosen recovery posture. If `Archive then delete` is chosen, the desired high-level result is:

- no available sandbox NAT Gateway
- no active sandbox ALBs for API/SSO
- no running or billable sandbox RDS/Redis compute
- no stopped sandbox EC2 instances with retained EBS/EIPs unless intentionally archived
- no Route53 records pointing at deleted sandbox endpoints
- Terraform updated or intentionally reconciled so the sandbox plan is clean

## External Operator Command Sequence

The commands below are for a human/operator shell. They are destructive AWS mutations and must not be run by Codex under the repository guardrails.

Set common variables:

```bash
export AWS_REGION=ap-southeast-2
export STAMP=20260429
```

### 1. Final recovery points

Create a final RDS snapshot by using the final-snapshot path during deletion:

```bash
aws rds delete-db-instance \
  --region "$AWS_REGION" \
  --db-instance-identifier sensor-sandbox \
  --final-db-snapshot-identifier "sensor-sandbox-final-${STAMP}"
```

Create a final Redis snapshot before deleting the replication group:

```bash
aws elasticache create-snapshot \
  --region "$AWS_REGION" \
  --replication-group-id smokealarmsandbox \
  --snapshot-name "smokealarmsandbox-final-${STAMP}"

aws elasticache wait snapshot-available \
  --region "$AWS_REGION" \
  --snapshot-name "smokealarmsandbox-final-${STAMP}"
```

Create EBS snapshots for stopped sandbox instance volumes:

```bash
aws ec2 create-snapshot --region "$AWS_REGION" --volume-id vol-0a89d230e0f94588b --description "OPT-001 final snapshot Odoo-sandbox ${STAMP}"
aws ec2 create-snapshot --region "$AWS_REGION" --volume-id vol-0d0627791dfcd6d78 --description "OPT-001 final snapshot SANDBOX-Kafka-mongo-Smoke ${STAMP}"
aws ec2 create-snapshot --region "$AWS_REGION" --volume-id vol-0a4f037bb9b336f0d --description "OPT-001 final snapshot smoke-sandbox-api-cron-server ${STAMP}"
aws ec2 create-snapshot --region "$AWS_REGION" --volume-id vol-01b3c948846ae36f8 --description "OPT-001 final snapshot Emqx-sandbox ${STAMP}"
aws ec2 create-snapshot --region "$AWS_REGION" --volume-id vol-0154f6eb5a20a6e99 --description "OPT-001 final snapshot openvpn-sandbox ${STAMP}"
```

### 2. Remove data stores

Delete Redis after the final snapshot is available:

```bash
aws elasticache delete-replication-group \
  --region "$AWS_REGION" \
  --replication-group-id smokealarmsandbox \
  --retain-primary-cluster false
```

Wait for RDS deletion to finish before removing dependent network resources:

```bash
aws rds wait db-instance-deleted \
  --region "$AWS_REGION" \
  --db-instance-identifier sensor-sandbox
```

### 3. Remove sandbox API/SSO public endpoints

Delete Route53 records that point directly at sandbox ALBs:

```bash
cat >/tmp/delete-sandbox-alb-dns.json <<'JSON'
{
  "Changes": [
    {
      "Action": "DELETE",
      "ResourceRecordSet": {
        "Name": "api-sandbox.sensorglobal.com.",
        "Type": "CNAME",
        "TTL": 300,
        "ResourceRecords": [
          { "Value": "smoke-sandbox-api-lb-621355096.ap-southeast-2.elb.amazonaws.com." }
        ]
      }
    },
    {
      "Action": "DELETE",
      "ResourceRecordSet": {
        "Name": "auth-sandbox.sensorglobal.com.",
        "Type": "CNAME",
        "TTL": 300,
        "ResourceRecords": [
          { "Value": "sandbox-sso-api-2060882049.ap-southeast-2.elb.amazonaws.com." }
        ]
      }
    }
  ]
}
JSON

aws route53 change-resource-record-sets \
  --hosted-zone-id Z03520551S3PY3J02L5ZN \
  --change-batch file:///tmp/delete-sandbox-alb-dns.json
```

Delete sandbox ALBs and target groups:

```bash
aws elbv2 delete-load-balancer \
  --region "$AWS_REGION" \
  --load-balancer-arn arn:aws:elasticloadbalancing:ap-southeast-2:747293622182:loadbalancer/app/Smoke-Sandbox-Api-LB/eb23b218ba5e1478

aws elbv2 delete-load-balancer \
  --region "$AWS_REGION" \
  --load-balancer-arn arn:aws:elasticloadbalancing:ap-southeast-2:747293622182:loadbalancer/app/sandbox-sso-api/c1b7b065cca73b1d

aws elbv2 wait load-balancers-deleted \
  --region "$AWS_REGION" \
  --load-balancer-arns \
    arn:aws:elasticloadbalancing:ap-southeast-2:747293622182:loadbalancer/app/Smoke-Sandbox-Api-LB/eb23b218ba5e1478 \
    arn:aws:elasticloadbalancing:ap-southeast-2:747293622182:loadbalancer/app/sandbox-sso-api/c1b7b065cca73b1d

aws elbv2 delete-target-group \
  --region "$AWS_REGION" \
  --target-group-arn arn:aws:elasticloadbalancing:ap-southeast-2:747293622182:targetgroup/Smoke-Sandbox-API-TG/0c5c8fcf29390601

aws elbv2 delete-target-group \
  --region "$AWS_REGION" \
  --target-group-arn arn:aws:elasticloadbalancing:ap-southeast-2:747293622182:targetgroup/sandbox-sso-api-tg/22235f2f5bf900ab
```

### 4. Remove NAT Gateway and route dependency

Delete the private default route to the sandbox NAT Gateway, then delete the NAT Gateway:

```bash
aws ec2 delete-route \
  --region "$AWS_REGION" \
  --route-table-id rtb-03fb390cce62724fc \
  --destination-cidr-block 0.0.0.0/0

aws ec2 delete-nat-gateway \
  --region "$AWS_REGION" \
  --nat-gateway-id nat-0f00d13d55f11c828
```

After the NAT Gateway reaches `deleted`, release its EIP:

```bash
aws ec2 release-address \
  --region "$AWS_REGION" \
  --allocation-id eipalloc-0121eef619a926133
```

### 5. Retire stopped sandbox EC2 instances and EIPs

Terminate stopped instances after EBS snapshots are complete:

```bash
aws ec2 terminate-instances \
  --region "$AWS_REGION" \
  --instance-ids \
    i-0074106adaa006f2e \
    i-0b71daa2680a16a5d \
    i-008e2acd8997ff315 \
    i-07af1c06915741068 \
    i-0c84b9424bc1ce4dd
```

Release their EIPs after termination and DNS cleanup/confirmation:

```bash
aws ec2 release-address --region "$AWS_REGION" --allocation-id eipalloc-04b3345e18629d6f4
aws ec2 release-address --region "$AWS_REGION" --allocation-id eipalloc-05cdec5ea2b6d6c5b
aws ec2 release-address --region "$AWS_REGION" --allocation-id eipalloc-028c41ffb5241e124
aws ec2 release-address --region "$AWS_REGION" --allocation-id eipalloc-0f3b645530195c938
aws ec2 release-address --region "$AWS_REGION" --allocation-id eipalloc-01577248aa180619e
```

Delete their EBS volumes after snapshots are complete and the instances are terminated:

```bash
aws ec2 delete-volume --region "$AWS_REGION" --volume-id vol-0a89d230e0f94588b
aws ec2 delete-volume --region "$AWS_REGION" --volume-id vol-0d0627791dfcd6d78
aws ec2 delete-volume --region "$AWS_REGION" --volume-id vol-0a4f037bb9b336f0d
aws ec2 delete-volume --region "$AWS_REGION" --volume-id vol-01b3c948846ae36f8
aws ec2 delete-volume --region "$AWS_REGION" --volume-id vol-0154f6eb5a20a6e99
```

### 6. Leave out of first pass

Do not delete these in the first pass unless a separate retirement decision is made:

- CloudFront distribution `EOKA8AFVFKKIR`
- S3 buckets `smoke-sandbox-api-lb-logs`, `smokealarmsandbox-angular`, `smokeapisandboxbucket`, `smokeapisandboxenv`
- `keychain-sandbox.sensorglobal.com` and any shared keychain infrastructure
- sandbox VPC, subnets, internet gateway, security groups, Secrets Manager secrets, log groups, alarms, and S3 buckets unless Terraform is updated in the same change set

## Approval Text Template

Use this text when asking for execution approval:

> Approve OPT-001 Sandbox fixed-cost cleanup using the `Archive then delete` posture. Approved scope: create final snapshots/backups, remove `sensor-sandbox`, `smokealarmsandbox`, sandbox NAT Gateway, sandbox API/SSO ALBs, approved stopped sandbox EC2 instances, their approved EBS volumes and EIPs, and update/remove sandbox DNS records. Keep shared keychain resources and production resources untouched. Changes will be applied externally, then Codex will verify read-only and record confirmed changes in `docs/applied-changes.md`.

## Open Questions

- Should sandbox be recoverable within hours, days, or only from Terraform plus backups?
- Should `api-sandbox` and `auth-sandbox` be deleted or parked on a static maintenance page?
- Is the sandbox CloudFront/S3 frontend still useful without backend/API/SSO?
- Is `smokealarmsandbox` Redis purely cache, or does any sandbox workflow expect durable Redis state?
- Should the stopped Odoo/Kafka/Mongo/EMQX/OpenVPN sandbox instances be retained as cold standby for a short grace period before final deletion?
