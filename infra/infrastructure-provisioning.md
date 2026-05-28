# Infrastructure Provisioning Method

**Date assessed:** 2026-04-12  
**Region:** ap-southeast-2  
**Account:** 747293622182

> **Historical status (updated 2026-04-27):** This note captures the state discovered on 2026-04-12, before the Terraform import work documented in `../applied-changes.md`. The current operational baseline is now `code/infra/terraform/`; keep this document as evidence of the legacy ClickOps origin and migration risks.

## Summary

**As of 2026-04-12, there was no user-authored IaC.** The entire SensorSyn/SensorGlobal AWS environment had been provisioned manually via the AWS Console ("ClickOps") and maintained that way since at least 2021. No Terraform state, no CDK stacks, no Pulumi config, and no user-authored CloudFormation templates existed for the core infrastructure at the time of assessment.

---

## Evidence

### CloudFormation — nothing user-authored for app infra

| Stack | Created | Notes |
|-------|---------|-------|
| `StackSet-AWS-QuickSetup-PatchPolicy-LA-*` (×5) | 2025 | **AWS-managed** — Systems Manager Quick Setup patch policies |
| `AWS-QuickSetup-PatchPolicy-LocalDeploymentRolesStack` | 2025-03-25 | **AWS-managed** |
| `NewRelic-Metric-Stream` | 2023-05-09 | New Relic telemetry integration — likely deployed from NR-provided template |
| `securityhubcsvmanagerstack-00` | 2024-11-05 | `ROLLBACK_COMPLETE` — failed and abandoned |
| `WAFSecurityAutomations` + sub-stack (us-east-1) | 2025-01-15 | AWS SOL-0006 WAF automation template — not bespoke |

No stacks cover VPCs, EC2, RDS, Kafka, EMQX, ALBs, security groups, or any application infrastructure.

### EC2 instances — Name tags only, no IaC markers

Tags on all instances (prod-kafka, Emqx-prod, smoke-prod-api-cron-server, etc.) are:
- `Name`: simple human-readable label
- `QSConfigName-5sbzn`: AWS Quick Setup patch policy assignment (AWS-managed)

No `aws:cloudformation:stack-name`, `terraform-*`, `cdktf-*`, or equivalent IaC tags.

### ALBs — going back to 2021, console-created

| ALB | Created |
|-----|---------|
| SmokeAPI-LB (prod) | 2021-09-23 |
| SmokedevelopmentLB | 2021-11-16 |
| SmokeqaLB | 2021-11-25 |
| Smoke-Sandbox-Api-LB | 2022-01-27 |
| keychain-production-ALB | 2022-03-09 |
| … | 2023–2024 |

### RDS — console-created

All RDS instances (sensor-prod, sensor-qa, sensor-sandbox, odoo-production, etc.) have only `Name` tags (and `devops-guru-default` on some Odoo DBs). No CF or IaC markers.

### VPCs — manually named

| VPC | CIDR | Notes |
|-----|------|-------|
| Smoke-VPC (prod) | 10.0.0.0/16 | Prod environment |
| Smoke-qa-VPC | 10.0.0.0/16 | |
| Smoke-Sandbox-VPC | 10.0.0.0/16 | |
| Smoke-development-VPC | 10.0.0.0/16 | |
| smoke-uat-vpc | 172.27.64.0/18 | |

### ASGs — created by CodeDeploy, not IaC

All Auto Scaling Groups are named `CodeDeploy_*` and were created automatically by CodeDeploy deployment groups when deployments were first triggered (2024–2025). They use manually-created launch templates (`prod-api-server-lt`, `Smoke-qa-api-lt`, etc.).

---

## What Does Exist

| Component | How provisioned |
|-----------|----------------|
| Application CI/CD pipeline | CodePipeline + CodeBuild + CodeDeploy — created in console, documented in `deployment-pipeline.md` |
| Launch templates | Console-created manually; `prod-api-server-lt` is on version 650 after years of manual edits |
| WAF (CloudFront) | AWS WAF Security Automations Solution template (standard AWS solution) |
| Patch management | AWS Systems Manager Quick Setup |
| Monitoring | New Relic (CloudFormation template from NR), Zabbix EC2 (`zabbix` instance) |

---

## Migration Implications

Because the original environment was ClickOps/manual, migrating to a new AWS account still requires careful discovery even though Terraform state now exists:

1. **Validate imported Terraform before reuse** — imported state reflects current resources, but not necessarily clean design intent.
2. **Manual discovery required** — every security group, subnet, route table, target group, listener rule, and IAM role must be inventoried to understand what to recreate.
3. **Write IaC during migration** — this is the opportunity to introduce Terraform (or CDK) so the new environment is reproducible.
4. **Hidden configuration** — settings like RDS parameter groups, ALB listener rules, EMQX broker config, Kafka broker settings, and EC2 UserData are stored only in the console/running state. They must be exported before the old account is decommissioned.
5. **Build environment** — CodeBuild buildspec is inline (not in repo); see `deployment-pipeline.md`. The S3 env bucket (`s3://smokeapienv/env`) contents must also be exported.

### Recommended approach for new environment

- Use **Terraform** (or AWS CDK) to define all infrastructure from day one in the new account.
- Treat the current environment as the reference spec — document every resource that needs to be recreated (see `docs/investigations/`).
- Consider using `aws-nuke` or manual inventory scripts to enumerate all resources in the current account before migration.
