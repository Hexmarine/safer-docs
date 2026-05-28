# AWS Account Baseline

- Date: 2026-04-09
- Scope: initial AWS CLI connectivity validation and high-level account context
- Related systems: AWS account `747293622182`, IAM, regional footprint, local investigation workflow

## Objective

Confirm that the AWS CLI is usable for read-only investigation work and capture the smallest useful account baseline for future documentation.

## Sources Used

- Repository files: `AGENTS.md`, `docs/README.md`, `docs/templates/investigation-template.md`
- AWS commands or consoles:
  - `aws sts get-caller-identity`
  - `aws configure list`
  - `aws configure get region`
  - `aws ec2 describe-regions --all-regions --query 'Regions[?OptInStatus!=\`not-opted-in\`].[RegionName,OptInStatus]' --output table`
  - `aws iam get-account-summary`
- External references: none

## Findings

- The AWS CLI is installed and working for read-only investigation when commands are run outside the local sandbox.
- The active local profile is `sensorsyn-mfa`.
- The default configured region is `ap-southeast-2`.
- The authenticated principal for this session is IAM user `arn:aws:iam::747293622182:user/peresada1@gmail.com`.
- The active AWS account is `747293622182`.
- The account currently reports standard commercial regions as enabled with `opt-in-not-required`, including `ap-southeast-2`, `ap-southeast-1`, `us-east-1`, `us-east-2`, `us-west-1`, `us-west-2`, `eu-west-1`, `eu-west-2`, `eu-west-3`, `eu-central-1`, `eu-north-1`, `ap-northeast-1`, `ap-northeast-2`, `ap-northeast-3`, `ap-south-1`, `ca-central-1`, and `sa-east-1`.
- The IAM account summary suggests a moderately mature account with:
  - 24 IAM users
  - 193 IAM roles
  - 139 managed policies
  - 10 IAM groups
  - 18 MFA devices, 11 currently in use
  - account-level MFA enabled

## Risks Or Constraints

- AWS access from this workspace depends on running commands outside the sandbox with approval.
- The principal currently in use is an IAM user rather than an assumed role, which may matter for audit expectations and least-privilege review later.
- The region list only confirms enabled regions, not where active workloads actually exist.
- IAM summary metrics are useful for orientation but do not identify which roles, users, or policies are high-risk or in active use.

## Cost Optimization Opportunities

- No direct cost findings from this baseline snapshot alone.
- Region scoping should help reduce noisy inventory work and avoid unnecessary analysis in unused regions once active workload regions are confirmed.

## Recommended Next Steps

- Determine which regions actually contain production resources, starting with `ap-southeast-2`.
- Build service inventory notes for core workload surfaces such as EC2, RDS, ECS, Lambda, S3, VPC, and CloudFront.
- Confirm whether this IAM user is the intended long-term investigation identity or whether a read-only assumed role should be preferred.
- Create stable topic-oriented reference docs under `docs/infra/` once regional and service patterns are clear.

## Open Questions

- Which enabled regions contain active production or shared services resources?
- Which services host the primary application workloads for SensorSyn?
- Is this account single-environment or does it contain multiple environments such as prod, staging, or dev?
- Are there organization-level controls, SCPs, or centralized logging accounts that should be included in future investigation scope?
