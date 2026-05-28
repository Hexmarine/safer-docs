# Terraform Drift Status

- Date: 2026-04-29
- Scope: `code/infra/terraform` local configuration, remote Terraform state, and read-only AWS drift checks
- Related systems: Terraform S3 backend, account-level AWS resources, prod, sandbox, decommissioned dev/qa/uat environments

## Objective

Establish the current Terraform baseline health after prior import work and later non-prod decommissioning, without making any AWS or Terraform state changes.

## Sources Used

- Repository files:
  - `code/infra/terraform/README.md`
  - `code/infra/terraform/environments/*`
  - `code/infra/terraform/.github/workflows/{plan,apply}.yml`
  - `docs/README.md`
  - `docs/applied-changes.md`
  - `docs/investigations/2026-04-20-infra-inventory.md`
- AWS commands:
  - `aws sts get-caller-identity`
  - `aws autoscaling describe-auto-scaling-groups`
  - `aws ec2 describe-launch-template-versions`
  - `aws logs describe-log-groups`
- Terraform commands:
  - `terraform validate`
  - `terraform plan -lock=false -input=false -no-color -detailed-exitcode`
  - `terraform state list`

## Findings

### Follow-up Reconciliation

After the initial drift snapshot, a local Terraform config reconciliation was applied for `prod`, `sandbox`, `dev`, and `qa`. No AWS resources were changed, and no Terraform apply or state mutation was run.

Current post-reconciliation status:

| Environment | Plan status |
|---|---|
| `account` | Clean from initial check |
| `prod` | Clean after external application of approved API ASG autoscaling change |
| `sandbox` | Clean after matching partial-decommissioned state |
| `dev` | Clean after deactivating decommissioned resource files |
| `qa` | Clean after deactivating decommissioned resource files |
| `uat` | Not reconciled in this pass |

Prod was first reconciled to live AWS, then updated again after approval to prepare the API autoscaling change (`min_size=2`, `max_size=5`, warm pool stopped). The user later confirmed the change was applied externally; read-only verification showed live AWS and Terraform are now aligned. The original drift findings below are retained as investigation history.

Terraform lives under `code/infra/terraform`, with the git repository root at `code/infra`. The local `code/infra` repository currently has no commits; both `terraform/` and `optimisation/` are untracked.

Terraform validation passes for all environment directories:

| Environment | Validation status |
|---|---|
| `account` | Valid |
| `dev` | Valid |
| `qa` | Valid |
| `sandbox` | Valid |
| `uat` | Valid |
| `prod` | Valid with warnings |
| `backend-bootstrap` | Valid |

Prod validation warnings are limited to redundant `ignore_changes` entries on Lambda `last_modified` attributes in generated Lambda resources. Terraform reports that `last_modified` is provider-controlled, so those ignores have no effect.

`account` plan is clean:

```text
No changes. Your infrastructure matches the configuration.
```

`prod` plan completes and shows one known in-place ASG drift:

```text
Plan: 0 to add, 1 to change, 0 to destroy.
```

The only prod change is `aws_autoscaling_group.prod_api_server`:

| Attribute | Live | Config wants |
|---|---:|---:|
| `max_size` | `3` | `5` |
| `min_size` | `3` | `2` |
| `warm_pool.pool_state` | `Running` | `Stopped` |

`sandbox` plan completes but is not clean:

```text
Plan: 2 to add, 3 to change, 0 to destroy.
```

The sandbox plan would recreate two CodeDeploy ASGs:

- `aws_autoscaling_group.sandbox_api_asg`
- `aws_autoscaling_group.sandbox_sso_asg`

Read-only AWS confirmation showed both ASG names are absent from live AWS:

```text
aws autoscaling describe-auto-scaling-groups ... -> []
```

The sandbox plan would also update two launch templates back to older AMIs/default versions:

| Resource | Live default | Config wants |
|---|---:|---:|
| `aws_launch_template.sandbox_api_lt` | version `186`, AMI `ami-0a41fe81bd2f8bbeb` | version `184`, AMI `ami-08b7a153c568c55fd` |
| `aws_launch_template.sandbox_sso_lt` | version `40`, AMI `ami-0369881db3a34806b` | version `39`, AMI `ami-0d40b8580df6291eb` |

The sandbox plan would also change `aws_cloudwatch_log_group.smoke_api_sandbox_pm2_out.retention_in_days` from live `365` to configured `14`.

`dev`, `qa`, and `uat` do not currently reach a normal plan. Each environment still has active `import` blocks for Secrets Manager secrets that no longer exist after decommissioning:

| Environment | Blocking missing imports |
|---|---|
| `dev` | `aws_secretsmanager_secret.sensor_dev_sso`, `aws_secretsmanager_secret.smoke_dev` |
| `qa` | `aws_secretsmanager_secret.sensor_qa`, `aws_secretsmanager_secret.sensor_qa_sso` |
| `uat` | `aws_secretsmanager_secret.sensor_uat`, `aws_secretsmanager_secret.sensor_uat_sso` |

The top-level Terraform README is stale: it says UAT import is pending, while `docs/applied-changes.md` records UAT import completion on 2026-04-17 and `docs/investigations/2026-04-20-infra-inventory.md` records UAT decommissioning on 2026-04-21.

The GitHub Actions apply workflow contains a likely control-flow bug: it runs `terraform plan -detailed-exitcode` before apply, but does not handle exit code `2`. GitHub Actions will normally fail a step on exit code `2`, even though the inline comment says exit code `2` should proceed to apply.

## Risks Or Constraints

- No AWS mutations were performed in this investigation.
- No Terraform state changes were performed.
- Plans were run with `-lock=false` for read-only drift inspection.
- The sandbox plan contains no destroy actions, but applying it as-is would recreate deleted ASGs and roll launch templates back to older defaults.
- Dev/qa/uat Terraform environments are not currently useful for normal drift review because stale import blocks fail planning before Terraform can summarize resource drift.
- The local repository is not committed, so there is no durable git history for the Terraform baseline in this checkout.

## Cost Optimization Opportunities

No new cost optimization was applied or recommended from these plan results.

The sandbox ASG absence is consistent with reduced compute cost, but Terraform config has not been reconciled with that desired state. If sandbox is intended to remain stopped or partially decommissioned, Terraform should be updated to encode that intent rather than recreating ASGs.

## Recommended Next Steps

1. Reconcile UAT separately or explicitly archive it as decommissioned.
2. Fix stale Terraform README content around UAT import/decommissioning.
3. Fix the GitHub Actions `apply.yml` `terraform plan -detailed-exitcode` handling before relying on CI apply.
4. Consider committing the Terraform baseline once the desired post-decommission state is clear.

## Open Questions

- Is sandbox meant to keep its RDS, Redis, ALBs, launch templates, and supporting resources but no app-tier ASGs?
- Were the sandbox ASGs deleted intentionally as part of cost reduction, or by CodeDeploy/deployment cleanup?
- Should dev/qa/uat Terraform directories remain as historical/import artifacts, or should they be archived out of active CI paths?
- Should prod `prod_api_server` ASG sizing and warm pool drift be reconciled to live AWS or to Terraform config?
