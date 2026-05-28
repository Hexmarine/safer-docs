# AWS Account Detach Analysis

- Date: 2026-04-28
- Scope: read-only analysis of detaching AWS account `747293622182` from its current parent AWS Organization while keeping the existing account and workloads
- Related systems: AWS Organizations, billing, account contacts, IAM, SCPs, Support, Cost Explorer, AWS Artifact, service quotas, migration planning

## Objective

Understand what is already known about the current account's parent organization, what AWS requires before a member account can become standalone, what risks this creates for the live SensorSyn environment, and the safest sequence for executing the detach when the business is ready.

No AWS mutation was performed during this investigation.

## Sources Used

- Repository files:
  - `AGENTS.md`
  - `docs/README.md`
  - `docs/gaps.md`
  - `docs/investigations/2026-04-09-aws-account-baseline.md`
  - `docs/investigations/2026-04-09-prod-external-account-migration-evaluation.md`
  - `docs/investigations/2026-04-10-migration-plan.md`
- AWS commands or consoles:
  - `aws sts get-caller-identity`
  - `aws configure get region`
  - `aws organizations describe-organization`
  - `aws organizations list-roots`
  - `aws organizations list-parents --child-id 747293622182`
  - `aws organizations list-policies-for-target --target-id 747293622182 --filter SERVICE_CONTROL_POLICY`
  - `aws organizations list-aws-service-access-for-organization`
  - `aws iam get-account-summary`
  - `aws account get-contact-information`
  - `aws account get-alternate-contact --alternate-contact-type BILLING|OPERATIONS|SECURITY`
  - `aws iam get-role --role-name OrganizationAccountAccessRole`
  - `aws iam list-roles`
  - `aws iam list-open-id-connect-providers`
  - `aws support describe-severity-levels --region us-east-1`
- External references:
  - AWS Organizations User Guide: leaving an organization from a member account, https://docs.aws.amazon.com/organizations/latest/userguide/orgs_manage_accounts_leave-as-member.html
  - AWS Organizations User Guide: removing a member account from an organization, https://docs.aws.amazon.com/organizations/latest/userguide/orgs_manage_accounts_remove.html
  - AWS Organizations User Guide: service control policies, https://docs.aws.amazon.com/organizations/latest/userguide/orgs_manage_policies_scps.html

## Findings

### Executive Summary

The current AWS account can plausibly be detached from its parent organization and kept as the production account, but this is a billing, governance, legal, and operational cutover, not just an AWS Organizations button click.

The account is currently a member account in AWS Organization `o-l2ps3sa79r`, whose management account is `872515272604`. The organization uses the `ALL` feature set and has Service Control Policies enabled. The current working assumption is that the new JV will keep account `747293622182` as a standalone AWS account and pay AWS directly, rather than joining a new AWS Organization immediately. New costs become the account owner's responsibility, organization-level guardrails stop applying, and organization-level service integrations or agreements may no longer cover the account.

The safest path is:

1. Complete standalone-account readiness in the member account: root ownership, payment method, phone verification, support plan, contact information, and billing access.
2. Export the parent-org context from the management account: OU path, directly and inherited SCPs, delegated-admin relationships, enabled trusted-access services, organization agreements, account tags, and billing/cost records.
3. Recreate replacement controls inside this account or in the destination organization before detach where possible.
4. Execute detach from the member account using `leave-organization`, assuming `organizations:LeaveOrganization` is allowed and standalone sign-up is complete.
5. Immediately validate billing, guardrails, security services, quota limits, DNS/domain renewal safety, Support access, and residual cross-account IAM access.

### Current AWS Account And Parent State

- Authenticated identity:
  - Account: `747293622182`
  - Principal: `arn:aws:iam::747293622182:user/peresada1@gmail.com`
  - Default region: `ap-southeast-2`
- Organization:
  - Organization ID: `o-l2ps3sa79r`
  - Management account: `872515272604`
  - Management account email observed from AWS Organizations: `AwsresellerenduserAU238@ingrammicro.com`
  - Feature set: `ALL`
  - Policy type enabled: `SERVICE_CONTROL_POLICY`
- Current IAM summary:
  - 24 users
  - 192 roles
  - 139 managed policies
  - 10 groups
  - 18 MFA devices, 11 in use
  - account MFA enabled
  - no account root access keys present
- Account contacts:
  - Primary contact information exists.
  - Alternate `BILLING`, `OPERATIONS`, and `SECURITY` contacts are not configured.
- Organization access roles:
  - `OrganizationAccountAccessRole` was not found.
  - `AWSServiceRoleForOrganizations` exists as the Organizations service-linked role.
- Identity providers:
  - No IAM OpenID Connect providers were listed.
- Support:
  - `aws support describe-severity-levels` returned `SubscriptionRequiredException`, indicating the current accessible support plan/API access is not Premium Support.

### Visibility Gaps From The Member Account

The current member-account IAM user can describe the organization, but cannot inspect several parent-side details:

| Check | Result | Detach relevance |
| --- | --- | --- |
| `organizations list-roots` | `AccessDeniedException` | Root and org policy topology need parent export |
| `organizations list-parents --child-id 747293622182` | `AccessDeniedException` | Current OU path is unknown from this account |
| `organizations list-policies-for-target` | `AccessDeniedException` | Directly attached SCPs are unknown |
| `organizations list-aws-service-access-for-organization` | `AccessDeniedException` | Trusted-access service dependencies are unknown |

This is normal for many member-account roles, but it means the detach decision should not be made from member-side evidence alone. The management account or reseller must provide an export of OU placement, inherited SCPs, organization integrations, delegated administrators, and account tags.

### AWS-Supported Detach Mechanisms

AWS supports two equivalent outcomes:

1. **Expected path: member-account initiated detach.**
   - Run from credentials in account `747293622182`.
   - CLI shape: `aws organizations leave-organization`.
   - Requires `organizations:LeaveOrganization`.
   - Can be blocked by an SCP.
2. **Fallback path: management-account initiated removal.**
   - Run from the management account `872515272604`.
   - CLI shape: `aws organizations remove-account-from-organization --account-id 747293622182`.
   - Requires `organizations:RemoveAccountFromOrganization` in the management account.

Both paths require the member account to be ready to operate as a standalone account. AWS may require completion of sign-up details before the detach can succeed: contact details, valid payment method, phone verification, and support plan selection.

### What Changes At Detach

The account is not closed and workloads should not stop merely because Organizations membership changes. However, several control planes change immediately:

| Area | Expected change |
| --- | --- |
| Billing | New charges after detach are paid by this account's payment method, not the parent management account. |
| Cost history | AWS docs state Cost Explorer visibility for historical member-period usage is lost from the detached account, though invoices/bills data remain available in specific ways. Export anything needed before detach. |
| SCPs and org policies | SCP restrictions stop applying. Users and roles may become more permissive if local IAM policies allowed actions that SCPs previously blocked. |
| Organization agreements | Agreements accepted by the organization management account no longer cover this account. Check AWS Artifact and legal requirements. |
| Trusted access / delegated admin | Organization service integrations can stop applying to this account. Security Hub, GuardDuty, Config, IAM Access Analyzer, Backup, SSO/IAM Identity Center, or other org-integrated services need explicit review. |
| Account tags | AWS docs state account tags attached through Organizations are deleted on removal. Export them first if they matter. |
| Service quotas | Quotas can change after leaving an organization. Snapshot and validate quota-sensitive services after detach. |
| Support | Support plan must be selected for standalone operation. The current Support API check does not show Premium Support access. |
| Parent IAM access | Organization-created or parent-created IAM roles are not automatically removed. Review and delete/disable any residual parent access after detach. |

### Existing SensorSyn-Specific Coupling To Protect

The account-detach path is less invasive than the earlier "move workloads to a different AWS account" migration plan, because EC2, RDS, ElastiCache, S3, Route 53 hosted zones, CloudFront, ACM, WAF, IAM, KMS, Terraform state, and domains remain in account `747293622182`.

That makes this preferable to a workload migration if the business goal is ownership separation without rebuilding production.

Still, the following existing SensorSyn items are specifically relevant:

- Route 53 Registrar and domains stay in this account, so payment method validity becomes critical before upcoming domain renewals.
- Organization SCP removal can expose latent IAM permissions. This account has 24 users, 192 roles, and 139 managed policies; local IAM review should happen before or immediately after detach.
- Existing cost and migration docs rely on Cost Explorer history. Export monthly cost/billing data before detachment in case historical Cost Explorer visibility changes.
- Terraform state and imported infrastructure remain in this account, so no infrastructure replacement is required for detachment itself.
- `docs/gaps.md` already notes the intended shape: AWS access is available and the source/parent account will detach while keeping this account.

## Risks Or Constraints

- **Do not execute detach casually.** It is a mutating AWS Organizations action and is forbidden under the current repository guardrails unless separately planned and approved outside this read-only investigation.
- **SCP unknowns are the biggest current evidence gap.** The organization reports SCPs enabled, but member-account access cannot list the OU path or attached policies.
- **Standalone billing readiness is not fully proven.** Primary contact exists, but payment method, phone verification, and support plan readiness could not be confirmed from CLI here.
- **Alternate contacts are missing.** Billing, operations, and security alternate contacts should be set before detach for operational continuity.
- **Support posture may be weaker after detach.** The Support API reports no Premium Support subscription for the accessible account context.
- **Cost Explorer history may become less useful from the detached account.** Export data first.
- **Org agreements may stop covering the account.** Contractual approval to detach is helpful, but AWS Artifact/legal agreement coverage still needs checking.
- **Local IAM may become more powerful after SCP removal.** Any permissions previously constrained by SCPs can become effective.

## Cost Optimization Opportunities

This detach analysis is not primarily a cost optimization exercise.

Potential financial effects to plan for:

- Parent/reseller billing charges stop covering new usage after detach.
- Support plan costs may move directly to this account.
- Marketplace subscriptions and vendor-linked AWS charges should be checked before and after detach.
- Cost history should be exported before detach so prior optimization baselines remain available.

## Recommended Next Steps

### Pre-Detach Evidence Pack

Ask the parent/reseller or management-account operator for:

- Current OU path for account `747293622182`.
- Direct and inherited SCPs, with JSON policy documents.
- Enabled trusted-access services for the organization.
- Delegated administrator registrations where this account is either the delegated admin or a managed member.
- Organization account tags.
- Organization agreements relevant to this account from AWS Artifact.
- Confirmation whether any consolidated billing, reseller, Savings Plan, reserved-instance, marketplace, or support arrangements are tied to the parent.

### Member Account Readiness

Before any detach attempt:

- Confirm root user email ownership and recovery path.
- Complete standalone account sign-up details if AWS prompts for them.
- Add/verify valid payment method.
- Verify phone number.
- Select appropriate support plan.
- Add alternate billing, operations, and security contacts.
- Export Cost Explorer, Bills, invoices, CUR if available, and recent service cost baselines.
- Snapshot service quotas for production-sensitive services: EC2, ELB, RDS, ElastiCache, VPC/NAT/EIP, CloudFront, Route 53, ACM, WAF, S3, SMS/messaging.

### Guardrail Replacement

Before or immediately after detach:

- Recreate essential SCP-style controls as local IAM permission boundaries, IAM policies, or new-organization SCPs.
- Review all admin users, access keys, and high-privilege roles.
- Confirm Security Hub, GuardDuty, Config, CloudTrail, IAM Access Analyzer, Backup, and logging are operating in standalone mode or under the new organization.
- Remove parent/reseller access roles that should no longer exist.

### Execution Path

Expected path: member account leaves and becomes standalone under JV-controlled billing.

```sh
aws organizations leave-organization
```

Use this if the member account has `organizations:LeaveOrganization`, no SCP blocks it, and standalone sign-up is complete. This matches the current JV assumption: no immediate new AWS Organization, no Ingram Micro billing after detach, and no workload migration.

Fallback path: parent management account removes the member.

```sh
aws organizations remove-account-from-organization --account-id 747293622182
```

Use this only if the member account cannot leave directly, for example because SCPs or member-account permissions block `leave-organization`. The member account still needs standalone readiness.

### Post-Detach Validation

Immediately after detach:

- Confirm `aws organizations describe-organization` from the member account no longer reports membership, or shows the expected new organization if the account is re-invited.
- Confirm billing console access, payment method, support plan, and alternate contacts.
- Confirm Route 53 domain AutoRenew and payment safety.
- Confirm production workloads are healthy: ALBs, EC2, RDS, ElastiCache, CloudFront, DNS, ACM, WAF, S3, EMQX, Kafka, Odoo, WordPress.
- Confirm security services and CloudTrail still log correctly.
- Confirm no unexpected IAM broadening from SCP removal.
- Confirm Cost Explorer/Bills visibility and preserve exported cost snapshots.

## Open Questions

- Confirm the standalone end state is long-term, not only an interim state before a future JV AWS Organization.
- Who has root-user access and authority to complete standalone account sign-up/payment details?
- What exact SCPs and org policies currently apply to account `747293622182`?
- Is this account a delegated administrator for any organization-level service?
- Which organization agreements currently cover this account via AWS Artifact?
- Are there reseller-specific billing, support, marketplace, Savings Plan, or RI arrangements that need replacement?
- Will `organizations:LeaveOrganization` be allowed from this member account, or will Ingram Micro need to remove the member account from the parent side?
