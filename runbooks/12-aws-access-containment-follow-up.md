# AWS Access Containment Follow-Up

## Purpose

Reduce remaining broad AWS entry points after the 2026-05-08 Appinventiv/OneLogin containment.

This runbook is intentionally operator-driven. Codex must not run the mutating commands in this repository; an authorized human should run them and then ask Codex to verify read-only.

## Current Evidence

Read-only prechecks on 2026-05-08 found:

- `Backup-user` no longer exists.
- SAML provider `onelogin` no longer exists.
- No CloudTrail `AssumeRoleWithSAML` events were found after the `onelogin` provider deletion window.

Follow-up verification on 2026-05-08 found:

- `MSP-user`:
  - no longer has attached policies
  - no longer has a console login profile
  - still exists, but has no access keys based on the earlier read-only precheck
- `sensorglobal-admin`:
  - no longer exists
- Current direct `AdministratorAccess` entities:
  - roles: `cloudfromation-template-role`, `AWS-QuickSetup-StackSet-Local-ExecutionRole`
  - groups: `Admin`, `SuperAdmin`
  - users: none

Use profile and region:

```bash
export AWS_PROFILE=sensorsyn-mfa
export AWS_REGION=ap-southeast-2
```

## Phase 1: Remove MSP User Admin Access

Status: completed externally and verified on 2026-05-08.

Impact:

- Removes a dormant direct admin user.
- At precheck time, `MSP-user` had no access keys, so the active risk was console password access.

Apply:

```bash
aws iam detach-user-policy \
  --user-name MSP-user \
  --policy-arn arn:aws:iam::aws:policy/AdministratorAccess

aws iam delete-login-profile \
  --user-name MSP-user
```

Optional full deletion after confirming the account is not required:

```bash
aws iam delete-user \
  --user-name MSP-user
```

Verify:

```bash
aws iam list-attached-user-policies \
  --user-name MSP-user

aws iam get-login-profile \
  --user-name MSP-user

aws iam get-user \
  --user-name MSP-user
```

Expected result:

- `list-attached-user-policies` returns an empty list if user still exists.
- `get-login-profile` returns `NoSuchEntity`.
- If the optional deletion was run, `get-user` returns `NoSuchEntity`.

## Phase 2: Remove Orphaned OneLogin Admin Role

Status: completed externally and verified on 2026-05-08.

Impact:

- Removes the old Appinventiv/OneLogin admin role path.
- Existing SAML provider is already deleted, so the role should no longer be assumable through OneLogin.

Apply:

```bash
aws iam detach-role-policy \
  --role-name sensorglobal-admin \
  --policy-arn arn:aws:iam::aws:policy/AdministratorAccess

aws iam delete-role \
  --role-name sensorglobal-admin
```

Verify:

```bash
aws iam get-role \
  --role-name sensorglobal-admin

aws iam list-entities-for-policy \
  --policy-arn arn:aws:iam::aws:policy/AdministratorAccess \
  --entity-filter Role
```

Expected result:

- `get-role` returns `NoSuchEntity`.
- `sensorglobal-admin` is absent from the AdministratorAccess role list.

## Phase 3: Review Remaining Admin Roles

Do not remove these blindly; confirm ownership first.

Current roles with `AdministratorAccess`:

- `cloudfromation-template-role`
  - Trusted by CloudFormation.
  - Last used 2025-01-15.
  - No inline policies.
  - No instance profiles.
- `AWS-QuickSetup-StackSet-Local-ExecutionRole`
  - Trusted by `AWS-QuickSetup-StackSet-Local-AdministrationRole`.
  - `RoleLastUsed` empty.
  - No inline policies.
  - No instance profiles.

If confirmed obsolete, use the same pattern:

```bash
aws iam detach-role-policy \
  --role-name ROLE_NAME \
  --policy-arn arn:aws:iam::aws:policy/AdministratorAccess

aws iam delete-role \
  --role-name ROLE_NAME
```

Verify:

```bash
aws iam get-role \
  --role-name ROLE_NAME
```

Expected result:

- `get-role` returns `NoSuchEntity`.

## Phase 4: Admin Groups Review

Status update: Tom admin reduction completed externally and verified on 2026-05-08.

Current broad admin groups:

- `Admin`
- `SuperAdmin`

Do not remove group policies until a replacement access model is agreed. Recommended approach:

1. Keep only current break-glass operators in `Admin`.
2. Remove stale or vendor accounts.
3. Replace day-to-day work with least-privilege roles.
4. Keep MFA mandatory for all console users.

Read-only review:

```bash
aws iam get-group \
  --group-name Admin

aws iam get-group \
  --group-name SuperAdmin
```

Current decision for Tom:

- `tom.mcevoy@sensorglobal.com` should stay for now.
- Current group membership is `ReadOnly-users`.
- Permanent `Admin` group membership was removed externally and verified.

Operator step:

```bash
aws iam remove-user-from-group \
  --group-name Admin \
  --user-name tom.mcevoy@sensorglobal.com
```

Verify:

```bash
aws iam list-groups-for-user \
  --user-name tom.mcevoy@sensorglobal.com
```

Expected result:

- `ReadOnly-users` remains.
- `Admin` is absent.

## Phase 5: Restore Access Analyzer

Current state:

- `ap-southeast-2` analyzer is disabled with reason `ORGANIZATION_DELETED`.
- `us-east-1` has no analyzer.

Create a standalone account analyzer after owner approval:

```bash
aws accessanalyzer create-analyzer \
  --region ap-southeast-2 \
  --analyzer-name sensor-account-analyzer \
  --type ACCOUNT
```

Verify:

```bash
aws accessanalyzer list-analyzers \
  --region ap-southeast-2
```

Expected result:

- Analyzer status is `ACTIVE`.

## Phase 6: Remove Remaining Appinventiv IAM User

Status update: completed externally and verified on 2026-05-08.

Current decision:

- `lokendra.saini@appinventiv.com` must be removed.
- The user was deleted externally.
- Read-only verification now returns `NoSuchEntity` for `get-user` and `get-login-profile`.

Operator steps:

```bash
aws iam delete-login-profile \
  --user-name lokendra.saini@appinventiv.com

aws iam deactivate-mfa-device \
  --user-name lokendra.saini@appinventiv.com \
  --serial-number arn:aws:iam::747293622182:mfa/sensor-project

aws iam delete-virtual-mfa-device \
  --serial-number arn:aws:iam::747293622182:mfa/sensor-project

aws iam delete-user \
  --user-name lokendra.saini@appinventiv.com
```

Verify:

```bash
aws iam get-user \
  --user-name lokendra.saini@appinventiv.com

aws iam get-login-profile \
  --user-name lokendra.saini@appinventiv.com

aws iam list-mfa-devices \
  --user-name lokendra.saini@appinventiv.com
```

Expected result:

- `get-user` and `get-login-profile` return `NoSuchEntity`.
- `list-mfa-devices` cannot return active devices for the deleted user.

## Phase 7: Reduce Database And SSM Blast Radius

Current findings:

- `sensor-prod` MySQL is private, but its security group allows `10.0.0.0/16`, so any prod VPC shell can network-reach it.
- `odoo-production` PostgreSQL is private, but it is reachable from `openvpn-5SG`.
- `openvpn-5SG` is public on UDP 1194, TCP 943, and TCP 443 from `0.0.0.0/0`.
- SSM port-forwarding is a valid operator path, but the shared `SSM_For_EC2` role has very broad permissions, including Secrets Manager read/write and EC2 full access.
- `SSM-SessionManagerRunShell` now streams session output to CloudWatch Logs group `/aws/ssm/session-manager`.

### Phase 7A: Remove Stale MySQL CIDR Rules

Status update: stale CIDR cleanup completed externally and verified on 2026-05-08. The broad `10.0.0.0/16` rule remains intentionally for the next controlled pass.

Target security group:

- `sg-0aa1e92c7b3a5c58e` / `DB-SG`

Keep the explicit application security group rules for now:

- `sg-0830a4cf088cc54be` / Production API SG
- `sg-0d544dd0240803824` / prod SSO API SG
- `sg-043324995d224dca0` / `PROD_AWS_CLIENT_VPN`, pending validation

Lowest-risk first removals:

```bash
aws ec2 revoke-security-group-ingress \
  --region ap-southeast-2 \
  --group-id sg-0aa1e92c7b3a5c58e \
  --protocol tcp \
  --port 3306 \
  --cidr 42.108.28.155/32

aws ec2 revoke-security-group-ingress \
  --region ap-southeast-2 \
  --group-id sg-0aa1e92c7b3a5c58e \
  --protocol tcp \
  --port 3306 \
  --cidr 20.0.0.0/22

aws ec2 revoke-security-group-ingress \
  --region ap-southeast-2 \
  --group-id sg-0aa1e92c7b3a5c58e \
  --protocol udp \
  --port 443 \
  --cidr 20.0.0.0/22
```

Do not remove `10.0.0.0/16` blindly. First confirm whether any non-API/non-SSO workloads still depend on it, especially `Smoke-prod-Keychain-app`, background jobs, or migration hosts. After explicit security group rules exist for every real DB client, remove the broad VPC rule:

```bash
aws ec2 revoke-security-group-ingress \
  --region ap-southeast-2 \
  --group-id sg-0aa1e92c7b3a5c58e \
  --protocol tcp \
  --port 3306 \
  --cidr 10.0.0.0/16
```

Verify:

```bash
aws ec2 describe-security-groups \
  --region ap-southeast-2 \
  --group-ids sg-0aa1e92c7b3a5c58e
```

Expected result:

- Stale `42.108.28.155/32` and `20.0.0.0/22` rules are absent.
- `10.0.0.0/16` is absent only after every required DB client has an explicit security group rule.

Current verified result:

- `42.108.28.155/32` and `20.0.0.0/22` are absent.
- UDP 443 from `20.0.0.0/22` is absent.
- `10.0.0.0/16` remains, so existing SSM investigation tunnels through prod VPC instances remain possible.

Terraform source to reconcile after the operator change:

- `code/infra/terraform/environments/prod/networking.tf`

### Phase 7B: Contain OpenVPN Access

Current OpenVPN path:

- Instance: `i-00abd001f35108630` / `openvpn-prod`
- Security group: `sg-0f4d3f24ee230e474` / `openvpn-5SG`
- Public ingress: UDP 1194, TCP 943, TCP 443 from `0.0.0.0/0`
- Odoo PostgreSQL ingress from this SG: `sg-08a034fb6f6fe51ac`

Preferred decision:

1. If OpenVPN is no longer required, retire it and remove the PostgreSQL ingress rule from `sg-08a034fb6f6fe51ac`.
2. If OpenVPN is still required, restrict `0.0.0.0/0` to approved office/admin CIDRs and audit OpenVPN users.

Contain Odoo PostgreSQL access from OpenVPN if approved:

```bash
aws ec2 revoke-security-group-ingress \
  --region ap-southeast-2 \
  --group-id sg-08a034fb6f6fe51ac \
  --ip-permissions 'IpProtocol=tcp,FromPort=5432,ToPort=5432,UserIdGroupPairs=[{GroupId=sg-0f4d3f24ee230e474}]'
```

If OpenVPN must stay, replace public ingress with approved CIDRs. Example shape, using placeholders:

```bash
aws ec2 revoke-security-group-ingress \
  --region ap-southeast-2 \
  --group-id sg-0f4d3f24ee230e474 \
  --protocol udp \
  --port 1194 \
  --cidr 0.0.0.0/0

aws ec2 authorize-security-group-ingress \
  --region ap-southeast-2 \
  --group-id sg-0f4d3f24ee230e474 \
  --protocol udp \
  --port 1194 \
  --cidr APPROVED_ADMIN_CIDR/32
```

Repeat for TCP 943 and TCP 443 if those ports are still required.

Terraform source to reconcile after the operator change:

- `code/infra/terraform/environments/prod/networking.tf`
- `code/infra/terraform/environments/prod/generated_prod.tf`

### Phase 7C: Split And Narrow `SSM_For_EC2`

Current attached policies:

- `AmazonSSMFullAccess`
- `AmazonEC2FullAccess`
- `SecretsManagerReadWrite`
- `CloudWatchFullAccess`
- `AWSCodeDeployFullAccess`
- `AmazonS3FullAccess`
- `AWSLambda_FullAccess`
- `AmazonInspector2FullAccess`
- `CloudWatchAgentServerPolicy`
- account policy `cloudwatchagent-policy`
- inline `AutoScalingLifecycleHookPolicy`

Current running prod instances with this profile include:

- `prod-kafka`
- `Smoke-prod-Keychain-app`
- `Odoo-production`
- `Emqx-prod`
- `wordpress-prod`
- `wordpress-sensor-insure`
- `prod-api-server`
- `prod-sso-api`

Recommended migration:

1. Create a new baseline EC2 role with AWS managed `AmazonSSMManagedInstanceCore` and the minimum CloudWatch Agent policy.
2. Create workload-specific roles for API, SSO, Keychain, Odoo, EMQX, Kafka, and WordPress.
3. Attach only the exact Secrets Manager ARNs and AWS actions each workload needs.
4. Move one instance group at a time.
5. Only after replacement roles are verified, detach broad policies from `SSM_For_EC2`.

Policies to remove from `SSM_For_EC2` after replacement roles are in place:

```bash
aws iam detach-role-policy \
  --role-name SSM_For_EC2 \
  --policy-arn arn:aws:iam::aws:policy/SecretsManagerReadWrite

aws iam detach-role-policy \
  --role-name SSM_For_EC2 \
  --policy-arn arn:aws:iam::aws:policy/AmazonEC2FullAccess

aws iam detach-role-policy \
  --role-name SSM_For_EC2 \
  --policy-arn arn:aws:iam::aws:policy/AmazonS3FullAccess

aws iam detach-role-policy \
  --role-name SSM_For_EC2 \
  --policy-arn arn:aws:iam::aws:policy/AWSLambda_FullAccess

aws iam detach-role-policy \
  --role-name SSM_For_EC2 \
  --policy-arn arn:aws:iam::aws:policy/AWSCodeDeployFullAccess
```

Terraform source to reconcile after the operator change:

- `code/infra/terraform/environments/account/iam.tf`
- `code/infra/terraform/environments/account/acm.tf`
- `code/infra/terraform/environments/account/route53.tf`
- `code/infra/terraform/environments/prod/generated_prod.tf`

### Phase 7D: Enable SSM Session Logging

Status update: completed externally and verified on 2026-05-08.

Current read-only finding:

- `SSM-SessionManagerRunShell` exists.
- Default document version is `2`.
- `cloudWatchLogGroupName` is `/aws/ssm/session-manager`.
- `cloudWatchStreamingEnabled` is `true`.
- `s3BucketName` is empty by design.
- CloudWatch log group `/aws/ssm/session-manager` exists with 90-day retention.

Recommended first step:

- Use CloudWatch Logs first, not S3, because it is enough for command/session forensics and has less bucket-policy surface.
- Use log group `/aws/ssm/session-manager`.
- Use 90-day retention initially.
- Use CloudWatch Logs default encryption for now. A customer-managed KMS key can be added later if required.

Operator steps:

```bash
export AWS_PROFILE=sensorsyn-mfa
export AWS_REGION=ap-southeast-2
export AWS_PAGER=""

aws logs create-log-group \
  --log-group-name /aws/ssm/session-manager \
  --no-cli-pager

aws logs put-retention-policy \
  --log-group-name /aws/ssm/session-manager \
  --retention-in-days 90 \
  --no-cli-pager
```

Create a local file `/tmp/ssm-session-manager-preferences.json`:

```json
{
  "schemaVersion": "1.0",
  "description": "Document to hold regional settings for Session Manager",
  "sessionType": "Standard_Stream",
  "inputs": {
    "s3BucketName": "",
    "s3KeyPrefix": "",
    "s3EncryptionEnabled": true,
    "cloudWatchLogGroupName": "/aws/ssm/session-manager",
    "cloudWatchEncryptionEnabled": false,
    "cloudWatchStreamingEnabled": true,
    "idleSessionTimeout": "20",
    "maxSessionDuration": "",
    "kmsKeyId": "",
    "runAsEnabled": false,
    "runAsDefaultUser": "",
    "shellProfile": {
      "windows": "",
      "linux": ""
    }
  }
}
```

Update Session Manager preferences:

```bash
aws ssm update-document \
  --name SSM-SessionManagerRunShell \
  --document-version '$LATEST' \
  --content file:///tmp/ssm-session-manager-preferences.json \
  --document-format JSON \
  --no-cli-pager
```

If AWS creates a new non-default document version, make it the default:

```bash
aws ssm describe-document \
  --name SSM-SessionManagerRunShell \
  --no-cli-pager

aws ssm update-document-default-version \
  --name SSM-SessionManagerRunShell \
  --document-version NEW_VERSION_FROM_DESCRIBE_DOCUMENT \
  --no-cli-pager
```

Then start and exit a short SSM session to a non-critical instance. Do not run sensitive commands in the test session.

Verify:

```bash
aws ssm get-document \
  --region ap-southeast-2 \
  --name SSM-SessionManagerRunShell

aws logs describe-log-groups \
  --region ap-southeast-2 \
  --log-group-name-prefix /aws/ssm/session-manager

aws logs describe-log-streams \
  --region ap-southeast-2 \
  --log-group-name /aws/ssm/session-manager \
  --order-by LastEventTime \
  --descending \
  --max-items 5
```

Expected result:

- `cloudWatchLogGroupName` is `/aws/ssm/session-manager`.
- Log group retention is 90 days.
- New SSM sessions produce command output logs.

Summary of owner-approved changes:

1. Narrow `sensor-prod` MySQL ingress to explicit application, SSO, EKS, and approved jump-host security groups.
2. Remove stale database CIDR rules such as broad VPC/client/dev ranges after validating replacement access.
3. Validate whether OpenVPN is still required; if it is, restrict ingress and audit users.
4. Split `SSM_For_EC2` into least-privilege instance roles per workload.
5. Enable SSM session logging to CloudWatch Logs or S3.

## Post-Change Verification

After each phase, ask Codex to verify read-only and update:

- `docs/investigations/2026-05-08-aws-admin-access-and-asg-scale-down.md`
- `docs/applied-changes.md`

Recommended CloudTrail spot checks:

```bash
aws cloudtrail lookup-events \
  --region us-east-1 \
  --lookup-attributes AttributeKey=EventName,AttributeValue=ConsoleLogin \
  --start-time START_TIME \
  --end-time END_TIME

aws cloudtrail lookup-events \
  --region us-east-1 \
  --lookup-attributes AttributeKey=EventName,AttributeValue=AssumeRoleWithSAML \
  --start-time START_TIME \
  --end-time END_TIME

aws cloudtrail lookup-events \
  --region us-east-1 \
  --lookup-attributes AttributeKey=EventName,AttributeValue=CreateAccessKey \
  --start-time START_TIME \
  --end-time END_TIME
```
