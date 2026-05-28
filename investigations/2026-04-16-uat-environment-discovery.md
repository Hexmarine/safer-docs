# UAT environment discovery

- Date: 2026-04-16
- Scope: Confirm whether a UAT environment still exists in the current AWS account and capture the minimum migration-relevant inventory
- Related systems: EC2, ALB, RDS, ElastiCache, CloudWatch, CodeDeploy, Secrets Manager

> **Historical status (updated 2026-04-27):** This note reflects UAT as discovered on 2026-04-16. It was later brought into the Terraform baseline and decommissioned on 2026-04-21; use `2026-04-20-infra-inventory.md` for current environment state.

## Objective

Determine whether UAT is still present as a real environment, or whether it had already been fully retired and could be ignored from the Terraform/migration baseline.

## Sources Used

- Repository files:
  - `code/infra/terraform/README.md`
  - `/home/yevgen/.copilot/session-state/fc394c54-0d63-4e21-b2f6-69a7fd2b39f1/plan.md`
- AWS commands or consoles:
  - `aws ec2 describe-instances --filters Name=tag:Name,Values=*uat*,*UAT*`
  - `aws elbv2 describe-load-balancers`
  - `aws elbv2 describe-target-groups`
  - `aws rds describe-db-instances`
  - `aws elasticache describe-replication-groups`
  - `aws cloudwatch describe-alarms`
  - `aws secretsmanager list-secrets`
  - `aws deploy list-applications`
  - `aws deploy list-deployment-groups`
- External references:
  - None

## Findings

UAT is still present in the account as a **partial but live environment**. It was previously excluded from the Terraform baseline, but that exclusion is no longer accurate.

Confirmed UAT inventory:

- **EC2:** 5 instances found by name
  - `uat-api-server` — `t3.medium` — **running**
  - `openvpn-uat` — `t2.micro` — stopped
  - `Emqx-uat` — `c5.large` — stopped
  - `odoo-uat-server` — `t3.medium` — stopped
  - `uat-kafka` — `t3.medium` — stopped
- **ALB:** 2 active load balancers
  - `smoke-uat-api`
  - `uat-sso-api-alb`
- **Target groups:** 2
  - `smoke-uat-api-tg` — HTTP 4553
  - `uat-sso-api` — HTTP 4100
- **RDS:** 1 MySQL instance
  - `smokealaramuat` — `db.t3.medium` — `MultiAZ=true` — `available`
- **ElastiCache:** 1 replication group
  - `smoke-uat-redis` — `available`
- **Secrets Manager:** 2 environment secrets
  - `sensor-uat`
  - `sensor-uat-sso`
- **CodeDeploy:** 2 applications / deployment groups
  - `sensor-uat-api-codedeploy` / `uat-api-deployment-group`
  - `uat-sso-codedeploy` / `uat-sso-dg`
- **CloudWatch alarms:** 20 env-scoped alarms plus 4 TargetTracking CodeDeploy alarms
  - Includes API/SOI target group alarms, RDS alarms, EC2 CPU/status alarms, and Lambda error alarms

Interpretation:

1. UAT is not fully retired; it still has a running API server and active ALBs.
2. The rest of the stack looks partly parked rather than deleted.
3. The migration baseline is incomplete if UAT is meant to survive into the target account.

## Risks Or Constraints

- UAT is currently **unmanaged in Terraform**; there is no `code/infra/terraform/environments/uat/`.
- Because only the API node is running while several supporting nodes are stopped, it is unclear whether UAT is intentionally dormant, partially broken, or only used sporadically.
- Some UAT alarms are in `ALARM` / `INSUFFICIENT_DATA`, which is consistent with a partially parked environment and makes health interpretation noisy.

## Cost Optimization Opportunities

- If UAT is no longer needed, it is a candidate for full retirement after explicit confirmation.
- If UAT is still needed, it should at minimum be:
  1. brought into Terraform,
  2. included in the health harness,
  3. evaluated for the same non-prod rightsizing / stop-start optimisation path as dev/qa/sandbox.

## Recommended Next Steps

1. Decide whether UAT is a keep / retire environment.
2. If keeping UAT, scaffold `code/infra/terraform/environments/uat/` and import core resources first (network, compute, data, load balancer).
3. Reuse `scripts/generate-missing-imports.py` to import UAT alarms and secrets once the env dir exists.
4. Add UAT to the migration checklist and health harness scope.

## Open Questions

- Is `uat-api-server` intentionally left running while the rest of UAT is stopped?
- Are the UAT ALBs still reachable by a real workflow, or are they just undeleted?
- Should `local` / `local-dev-sso` secrets remain outside the migration baseline, or do they need a separate developer-environment story?
