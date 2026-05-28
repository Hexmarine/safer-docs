# AWS Admin Access And ASG Scale-Down

- Date: 2026-05-08
- Scope: Production AWS account `747293622182`, API/SSO outage window on 2026-05-07
- Related systems: IAM, OneLogin SAML, CloudTrail, Auto Scaling, ALB target groups

## Objective

Understand who changed production API/SSO capacity, how that identity logged in, and which AWS principals currently have broad access capable of repeating the change.

## Sources Used

- Repository files:
  - `docs/applied-changes.md`
- AWS commands or consoles:
  - `cloudtrail lookup-events` for `UpdateAutoScalingGroup`, `ConsoleLogin`, `Backup-user`, and access key usage
  - `iam get-role`, `iam list-attached-role-policies`, `iam list-entities-for-policy`
  - `iam get-group`, `iam list-users`, `iam list-access-keys`, `iam get-access-key-last-used`
- External references:
  - None

## Findings

### ASG scale-down cause

CloudTrail records two `autoscaling.amazonaws.com:UpdateAutoScalingGroup` events from the same session:

| Local time | Resource | Change |
| --- | --- | --- |
| 2026-05-07 21:11:05 | `CodeDeploy_Smoke-API_d-NNRF41DGD` | `minSize=0`, `maxSize=0`, `desiredCapacity=0` |
| 2026-05-07 21:11:32 | `CodeDeploy_sso-api-dg_d-QKLSZP50D` | `minSize=0`, `maxSize=0`, `desiredCapacity=0` |

The user identity for both events was:

- Principal type: `AssumedRole`
- Assumed role ARN: `arn:aws:sts::747293622182:assumed-role/sensorglobal-admin/chaitanya.sharma@appinventiv.com`
- IAM role: `arn:aws:iam::747293622182:role/sensorglobal-admin`
- Source IP: `103.108.4.121`
- User agent: Chrome on Linux
- `sessionCredentialFromConsole`: `true`
- `mfaAuthenticated`: `false`

No direct ALB listener or target-group mutation was identified as the root change. Later `elasticloadbalancing.amazonaws.com:DeregisterTargets` events were invoked by `autoscaling.amazonaws.com` through `AWSServiceRoleForAutoScaling` after the ASG desired capacity was set to zero.

### How the identity logged in

CloudTrail `ConsoleLogin` shows successful federated AWS Console sign-ins:

| Local time | Principal | Method |
| --- | --- | --- |
| 2026-05-07 20:22:15 | `chaitanya.sharma@appinventiv.com` | SAML via `arn:aws:iam::747293622182:saml-provider/onelogin` |
| 2026-05-07 20:37:09 | `chaitanya.sharma@appinventiv.com` | SAML via `arn:aws:iam::747293622182:saml-provider/onelogin` |

CloudTrail additional data:

- `ConsoleLogin`: `Success`
- `MFAUsed`: `No`
- SAML provider: `onelogin`
- Login target: `https://console.aws.amazon.com/console/home`

The AWS SAML provider metadata points at:

- Entity ID: `https://onelogin.appinventiv.com/realms/appinventiv-aws`
- SSO endpoint: `https://onelogin.appinventiv.com/realms/appinventiv-aws/protocol/saml`

AWS IAM only shows that OneLogin can federate into `sensorglobal-admin`. The actual human/group membership that grants this access is controlled in OneLogin and is not visible from AWS IAM.

### New direct admin user created during session

The same `sensorglobal-admin` console session created a new IAM user and long-lived access key before the ASG scale-down:

| Local time | Event | Details |
| --- | --- | --- |
| 2026-05-07 20:38:02 | `CreateUser` | Created `Backup-user` |
| 2026-05-07 20:38:02 | `AttachUserPolicy` | Attached `arn:aws:iam::aws:policy/AdministratorAccess` to `Backup-user` |
| 2026-05-07 20:38:21 | `CreateAccessKey` | Created active key `AKIA237RLX6TDZFDABN2` |
| 2026-05-07 20:38:21 | `TagUser` | Added tag key `AKIA237RLX6TDZFDABN2`, value `backup user` |

Initial IAM state observed before containment:

- `Backup-user` has direct `AdministratorAccess`.
- `Backup-user` has no group memberships.
- `Backup-user` has one active access key: `AKIA237RLX6TDZFDABN2`.
- `get-access-key-last-used` reports last use at 2026-05-07 21:36 local time, service `iam`, region `us-east-1`.
- CloudTrail shows the key was used from `103.108.4.121` with AWS CLI for repeated `sts:GetCallerIdentity` checks between 2026-05-07 20:47 and 21:36 local time.

No mutating API calls from this access key were identified in the first CloudTrail lookup, but the key is active and has administrator privileges.

### Principals with AdministratorAccess

AWS IAM initially showed these direct attachments to `arn:aws:iam::aws:policy/AdministratorAccess`.

Roles:

- `sensorglobal-admin`
  - Trusted by SAML provider `arn:aws:iam::747293622182:saml-provider/onelogin`
  - Max session duration: 1 hour
- `cloudfromation-template-role`
  - Trusted by `cloudformation.amazonaws.com`
  - Last used: 2025-01-15
- `AWS-QuickSetup-StackSet-Local-ExecutionRole`
  - Trusted by `arn:aws:iam::747293622182:role/AWS-QuickSetup-StackSet-Local-AdministrationRole`
  - RoleLastUsed empty in `get-role`

Direct IAM users:

- `Backup-user`
  - Created 2026-05-07 20:38 local time
  - Active access key present
- `MSP-user`
  - No access keys observed via `list-access-keys`
  - Password last used 2025-11-20

IAM groups:

- `Admin`
  - Members observed: `tom.mcevoy@sensorglobal.com`, `peresada@gmail.com`, `kristyn.heywood@syncom.com.au`, `lokendra.saini@appinventiv.com`, `Richard.Heywood@syncom.com.au`, `peresada1@gmail.com`
- `SuperAdmin`
  - Members observed: `peresada@gmail.com`

The OneLogin SAML membership for `sensorglobal-admin` must be reviewed in OneLogin because AWS does not expose the external IdP assignment list.

Follow-up containment removed `Backup-user`, removed direct admin access and console login for `MSP-user`, and deleted the orphaned `sensorglobal-admin` role. The current post-containment state is listed under Remaining AWS entry points.

### Containment verification

The following externally applied containment actions were verified in CloudTrail. Codex did not perform these mutations.

| Local time | Event | Principal | Details |
| --- | --- | --- | --- |
| 2026-05-08 08:00:34 | `UpdateAccessKey` | `peresada1@gmail.com` | Set `Backup-user` key `AKIA237RLX6TDZFDABN2` to `Inactive` |
| 2026-05-08 08:00:46 | `DeleteSAMLProvider` | `peresada1@gmail.com` | Deleted `arn:aws:iam::747293622182:saml-provider/onelogin` |
| 2026-05-08 08:00:57 | `RemoveUserFromGroup` | `peresada1@gmail.com` | Removed `lokendra.saini@appinventiv.com` from IAM group `Admin` |
| 2026-05-08 08:01:08 | `DetachUserPolicy` | `peresada1@gmail.com` | Detached `AdministratorAccess` from `Backup-user` |
| 2026-05-08 08:01:18 | `DeleteAccessKey` | `peresada1@gmail.com` | Deleted `AKIA237RLX6TDZFDABN2` for `Backup-user` |
| 2026-05-08 08:01:27 | `DeleteUser` | `peresada1@gmail.com` | Deleted `Backup-user` |
| 2026-05-08 08:22:54 | `DetachUserPolicy` | `peresada1@gmail.com` | Detached `AdministratorAccess` from `MSP-user` |
| 2026-05-08 08:25:27 | `DeleteLoginProfile` | `peresada1@gmail.com` | Deleted console login profile for `MSP-user` |
| 2026-05-08 08:29:22 | `DetachRolePolicy` | `peresada1@gmail.com` | Detached `AdministratorAccess` from `sensorglobal-admin` |
| 2026-05-08 08:29:33 | `DeleteRole` | `peresada1@gmail.com` | Deleted `sensorglobal-admin` |

The containment session had `mfaAuthenticated=true`.

Read-only current-state checks after containment:

- `iam list-saml-providers` no longer shows `arn:aws:iam::747293622182:saml-provider/onelogin`.
- The only remaining SAML provider is `arn:aws:iam::747293622182:saml-provider/DEV-VPN`.
- `iam get-user --user-name Backup-user` returns `NoSuchEntity`.
- `iam list-attached-user-policies --user-name MSP-user` returns no attached policies.
- `iam get-login-profile --user-name MSP-user` returns `NoSuchEntity`.
- `iam get-role --role-name sensorglobal-admin` returns `NoSuchEntity`.
- No successful `ConsoleLogin` or `AssumeRoleWithSAML` events were observed after containment.
- `CreateUser` and `CreateAccessKey` checks in the post-incident window found no new events after the containment period.

An additional pre-containment Appinventiv SAML login was found:

| Local time | Principal | Method |
| --- | --- | --- |
| 2026-05-08 03:30:24 | `aniket2@appinventiv.com` | SAML via deleted provider `onelogin`, assuming `sensorglobal-admin`, `MFAUsed=No` |

This confirms OneLogin/Appinventiv assignment to `sensorglobal-admin` was broader than one person before the provider was deleted.

### Detailed activity timeline

All times below are Australia/Sydney local time.

| Time | Actor | Event | Detail |
| --- | --- | --- | --- |
| 2026-05-07 20:22:15 | `chaitanya.sharma@appinventiv.com` | `ConsoleLogin` | Successful SAML console login through `onelogin`, `MFAUsed=No`, source IP `103.108.4.121` |
| 2026-05-07 20:37:09 | `chaitanya.sharma@appinventiv.com` | `ConsoleLogin` | Successful SAML console login through `onelogin`, `MFAUsed=No`, source IP `103.108.4.121` |
| 2026-05-07 20:38:02 | `chaitanya.sharma@appinventiv.com` | `CreateUser` | Created IAM user `Backup-user` |
| 2026-05-07 20:38:02 | `chaitanya.sharma@appinventiv.com` | `AttachUserPolicy` | Attached `AdministratorAccess` to `Backup-user` |
| 2026-05-07 20:38:21 | `chaitanya.sharma@appinventiv.com` | `CreateAccessKey` | Created access key `AKIA237RLX6TDZFDABN2` for `Backup-user` |
| 2026-05-07 20:38:21 | `chaitanya.sharma@appinventiv.com` | `TagUser` | Tagged `Backup-user`; tag value observed as `backup user` |
| 2026-05-07 20:47-21:36 | `Backup-user` | AWS CLI read activity | Used `AKIA237RLX6TDZFDABN2` from `103.108.4.121`; observed STS and IAM inventory/read calls, including `GetAccountAuthorizationDetails` |
| 2026-05-07 21:11:05 | `chaitanya.sharma@appinventiv.com` | `UpdateAutoScalingGroup` | Set API ASG `CodeDeploy_Smoke-API_d-NNRF41DGD` to zero capacity |
| 2026-05-07 21:11:32 | `chaitanya.sharma@appinventiv.com` | `UpdateAutoScalingGroup` | Set SSO ASG `CodeDeploy_sso-api-dg_d-QKLSZP50D` to zero capacity |
| 2026-05-08 03:30:24 | `aniket2@appinventiv.com` | `ConsoleLogin` | Successful SAML console login through `onelogin`, `MFAUsed=No`, source IP `103.106.232.53` |
| 2026-05-08 07:57:55-08:29:38 | `aniket2@appinventiv.com` | Assumed-role console activity | Pre-existing `sensorglobal-admin` session made read-only console/API calls after provider deletion; latest observed call was `ListManagedNotificationEvents` at 08:29:38 with `AccessDenied` |
| 2026-05-08 08:00:34 | `peresada1@gmail.com` | `UpdateAccessKey` | Set `Backup-user` access key to `Inactive`; session `mfaAuthenticated=true`, source IP `122.151.80.69` |
| 2026-05-08 08:00:46 | `peresada1@gmail.com` | `DeleteSAMLProvider` | Deleted `onelogin` SAML provider |
| 2026-05-08 08:00:57 | `peresada1@gmail.com` | `RemoveUserFromGroup` | Removed `lokendra.saini@appinventiv.com` from IAM group `Admin` |
| 2026-05-08 08:01:08 | `peresada1@gmail.com` | `DetachUserPolicy` | Detached `AdministratorAccess` from `Backup-user` |
| 2026-05-08 08:01:18 | `peresada1@gmail.com` | `DeleteAccessKey` | Deleted `AKIA237RLX6TDZFDABN2` |
| 2026-05-08 08:01:27 | `peresada1@gmail.com` | `DeleteUser` | Deleted `Backup-user` |
| 2026-05-08 08:22:54 | `peresada1@gmail.com` | `DetachUserPolicy` | Detached `AdministratorAccess` from `MSP-user` |
| 2026-05-08 08:25:27 | `peresada1@gmail.com` | `DeleteLoginProfile` | Deleted `MSP-user` console login profile |
| 2026-05-08 08:29:22 | `peresada1@gmail.com` | `DetachRolePolicy` | Detached `AdministratorAccess` from `sensorglobal-admin` |
| 2026-05-08 08:29:33 | `peresada1@gmail.com` | `DeleteRole` | Deleted `sensorglobal-admin` |
| 2026-05-08 08:32:47-08:33:42 | hidden IAM username | `ConsoleLogin` failures | Three failed AWS console login attempts from `103.106.232.53`; CloudTrail error: `No username found in supplied account`; `ConsoleLogin=Failure` |

Post-role-deletion checks through 2026-05-08 08:42:49 local time found no successful `ConsoleLogin`, no `AssumeRoleWithSAML`, no `CreateUser`, no `CreateAccessKey`, no `AttachUserPolicy`, no `UpdateAutoScalingGroup`, and no further `chaitanya.sharma@appinventiv.com` or `Backup-user` events. The only `aniket2@appinventiv.com` event at or after role deletion was the 08:29:38 read-only notification call that returned `AccessDenied`.

### Current effective access snapshot

Snapshot time: 2026-05-08 09:13 AEST.

User-confirmed expected human access:

| User | Effective access | Console login | MFA | Active keys | Notes |
| --- | --- | --- | --- | --- | --- |
| `peresada1@gmail.com` | `Admin` group, including `AdministratorAccess` | Yes | 1 device | `AKIA237RLX6TEF65XKOL`, last used 2026-05-08 07:24 local via STS | Expected operator; active long-lived admin key remains |
| `kristyn.heywood@syncom.com.au` | `Admin` group, including `AdministratorAccess` | Yes | 1 device | None | Expected |
| `Richard.Heywood@syncom.com.au` | `Admin` group, including `AdministratorAccess` | Yes | 0 devices | None | Expected, but MFA is not configured |

Additional human IAM users with effective access:

| User | Effective access | Console login | MFA | Active keys | Classification |
| --- | --- | --- | --- | --- | --- |
| `peresada@gmail.com` | `Admin` and `SuperAdmin`, including `AdministratorAccess` plus billing/network/database/power-user policies | Yes | 2 devices | None | Confirmed expected operator on 2026-05-08 |
| `tom.mcevoy@sensorglobal.com` | `ReadOnly-users` | Yes | 1 device | None | Kept for now; removed from permanent `Admin` on 2026-05-08 |

Former or dormant human-like users without current permissions:

| User | Console login | MFA | Groups/policies | Active keys | Effective access |
| --- | --- | --- | --- | --- | --- |
| `lokendra.saini@appinventiv.com` | No; user deleted | N/A | N/A | N/A | Removed externally and verified on 2026-05-08 |
| `MSP-user` | No | 0 devices | None | None | No current effective access observed |

Service or automation users with active effective access:

| User | Credential path | Effective permissions | Notes |
| --- | --- | --- | --- |
| `bitrise` | Active key `AKIA237RLX6TJU6UJY2C`; no console login | Custom `Bitrise` policy: `s3:GetObject` and `s3:PutObject` on `smokeapidevelopmentenv/bitrise-android-version/*` | Last-used data returned `N/A`; confirm if still needed |
| `zabbix_monitoring` | Active key `AKIA237RLX6TOGJMMXGM`; no console login | Custom `Zabbix-AWS-Monitoring`: CloudWatch describe/get/list, EC2 describe, RDS describe | Last used 2026-04-22 07:49 local |
| `security-hub-finding` | Console login; no active keys | Group policies `AWSSecurityHubFullAccess`, `AmazonS3FullAccess`, `AmazonInspector2FullAccess`; inline `kms-custom-policy` | This is a broad service-style account with console login; classify ownership and need |

Users with policies but no current credential path observed:

| User | Policies/groups | Console login | Active keys | Current state |
| --- | --- | --- | --- | --- |
| `cloudtrail-CMK-User` | Direct `AmazonS3FullAccess`, `AWSCloudTrail_FullAccess` | No | None | No current credential path observed |
| `lambda-fun-sandbox` | Direct `AWSLambda_FullAccess` | No | None; inactive keys present | No current credential path observed |
| `SNS_S3_USER_QA` | Direct `AmazonSNSFullAccess`, `AmazonS3FullAccess` | No | None; inactive key present | No current credential path observed |
| `secret-manager-local-sso` | Inline `local-dev-sso-read-write` | No | None | No current credential path observed |
| `odoo-dev` | `ReadOnly-users` group | No | None | No current credential path observed |

Federated or external role access:

| Principal path | Effective permissions | Current status |
| --- | --- | --- |
| Root account | Root password present, root MFA enabled, root access keys absent | Break-glass access exists and is MFA-protected |
| `cloudfromation-template-role` | Direct `AdministratorAccess`; trusted by `cloudformation.amazonaws.com` | Last used 2025-01-15 |
| `AWS-QuickSetup-StackSet-Local-ExecutionRole` | Direct `AdministratorAccess`; trusted by local Quick Setup administration role | No `RoleLastUsed` value observed |
| `NewRelicInfrastructure-Integrations` | AWS `ReadOnlyAccess` plus inline `NewRelicBudget`; trusted external account `754728514883` with external ID | Last used 2026-05-08 09:03 local |
| `vanta-auditor` | `SecurityAudit` plus `VantaAdditionalPermissions`; trusted external account `956993596390` with external ID | Last used 2026-05-08 09:00 local |
| `Audits-Automation-Scripts-Role-Sensor-Global` | `Audits-Automation-Scripts-Policy`; trusted external account root `043210536673` with no external ID condition | Last used 2025-11-13; needs ownership review |
| `SaferOpsGitHubImagePublisherRole` | Scoped ECR image publishing to Safer Ops repositories | GitHub OIDC limited to `repo:SaferHomesAu/safer-ops:environment:production`; last used 2026-05-06 |
| `DEV-VPN` SAML provider | No IAM role currently trusts this provider based on role trust query | Provider remains registered but no current AWS role path observed |

### Remaining AWS entry points

#### Root account

`iam get-account-summary` shows:

- Root account password exists: `AccountPasswordPresent=1`.
- Root MFA is enabled: `AccountMFAEnabled=1`.
- Root access keys are absent: `AccountAccessKeysPresent=0`.

#### Direct IAM administrator access

Current direct `AdministratorAccess` attachments:

Roles:

- `cloudfromation-template-role`
  - Trusted by `cloudformation.amazonaws.com`.
  - Still has `AdministratorAccess`.
  - Last used 2025-01-15.
- `AWS-QuickSetup-StackSet-Local-ExecutionRole`
  - Trusted by local Quick Setup administration role.
  - Still has `AdministratorAccess`.
  - `RoleLastUsed` empty in `get-role`.

Users:

- No direct IAM users currently have `AdministratorAccess`.
- `MSP-user` still exists, but current read-only checks show no attached policies, no access keys, and no console login profile.

Groups:

- `Admin`
  - Still has `AdministratorAccess`.
  - Current members: `peresada@gmail.com`, `kristyn.heywood@syncom.com.au`, `Richard.Heywood@syncom.com.au`, `peresada1@gmail.com`.
  - `tom.mcevoy@sensorglobal.com` and `lokendra.saini@appinventiv.com` are no longer members.
- `SuperAdmin`
  - Still has `AdministratorAccess`.
  - Current member: `peresada@gmail.com`.

The remaining Appinventiv IAM user `lokendra.saini@appinventiv.com` was deleted externally and verified by read-only `NoSuchEntity` responses on 2026-05-08.

#### Long-lived active IAM access keys

Active access keys observed after containment:

| User | Access key | Last used | Permissions summary |
| --- | --- | --- | --- |
| `peresada1@gmail.com` | `AKIA237RLX6TEF65XKOL` | 2026-05-08 07:24 local, STS | In `Admin` group |
| `bitrise` | `AKIA237RLX6TJU6UJY2C` | No last-used service/date reported | `s3:GetObject`/`s3:PutObject` on `smokeapidevelopmentenv/bitrise-android-version/*` |
| `zabbix_monitoring` | `AKIA237RLX6TOGJMMXGM` | 2026-04-22 07:49 local, CloudWatch | Read-only CloudWatch, EC2, and RDS describe/list/get |

Inactive keys remain on several service users. They are not active entry points but should be deleted after ownership is confirmed.

#### Federated and external-account roles

SAML:

- `onelogin` provider has been deleted.
- `DEV-VPN` SAML provider remains, but no IAM role currently trusts `DEV-VPN` based on the role trust query.

OIDC:

- EKS OIDC provider exists for `safer-ops-prod`; this is expected for IRSA.
- GitHub OIDC provider `token.actions.githubusercontent.com` exists.
- `SaferOpsGitHubImagePublisherRole` trusts only `repo:SaferHomesAu/safer-ops:environment:production` and has ECR image publishing permissions scoped to `safer-ops-api` and `safer-ops-web`.

Cross-account AWS principals:

- `Audits-Automation-Scripts-Role-Sensor-Global`
  - Trusted by external account root `arn:aws:iam::043210536673:root`.
  - No external ID condition observed.
  - Permissions include audit reads plus writes to CloudWatch Logs, SNS publish, Config `PutEvaluations`, and IAM credential report generation.
- `NewRelicInfrastructure-Integrations`
  - Trusted by external account `754728514883` with external ID `3928230`.
  - Attached policy: AWS managed `ReadOnlyAccess`.
- `vanta-auditor`
  - Trusted by external account `956993596390` with external ID.
  - Attached policies: AWS managed `SecurityAudit` plus `VantaAdditionalPermissions`, whose current document is deny-only for three actions.

IAM Access Analyzer:

- `ap-southeast-2` has `ConsoleAnalyzer-c3a32e74-a56b-437f-86fc-e3e8b0e788a0`, but it is `DISABLED` with reason `ORGANIZATION_DELETED`.
- No analyzer was found in `us-east-1`.

### Database And Instance Entry Points

No Microsoft SQL Server RDS engine was observed in the current RDS inventory. The SQL entry points observed are MySQL and PostgreSQL:

| Database | Engine | Public | Network paths observed |
| --- | --- | --- | --- |
| `sensor-prod` / `smokealarmprod` | MySQL 8.0.44 on port 3306 | No | API server SG, SSO API SG, `PROD_AWS_CLIENT_VPN` SG, and broad prod VPC CIDR `10.0.0.0/16`. Stale `20.0.0.0/22` and `42.108.28.155/32` CIDR rules were removed externally and verified on 2026-05-08 |
| `odoo-production` | PostgreSQL 14.17 on port 5432 | No | Odoo EC2 SG, `openvpn-5SG`, and `10.0.4.167/32` labelled `Prod PowerBI Gateway` |

Important access observations:

- The databases are not internet-public, but `sensor-prod` MySQL allows all of `10.0.0.0/16`; any shell access on a running prod VPC instance or EKS node can network-reach it.
- Stale `sensor-prod` MySQL ingress from `42.108.28.155/32`, `20.0.0.0/22`, and UDP 443 from `20.0.0.0/22` was removed externally and verified on 2026-05-08.
- `openvpn-5SG` is public on UDP 1194, TCP 943, and TCP 443 from `0.0.0.0/0`, and can reach Odoo PostgreSQL. This is a major external entry point to validate or retire.
- No AWS Client VPN endpoints were observed, so the remaining `PROD_AWS_CLIENT_VPN` security group path should still be validated. The stale `20.0.0.0/22` CIDR database rules were removed.
- Existing local DB runbooks use SSM port forwarding through an EC2 target. That pattern is operationally useful but means SSM shell/session access can become DB network access.
- No active SSM sessions were observed during the check. Recent SSM history did include sessions by `chaitanya.sharma@appinventiv.com` and `aniket2@appinventiv.com` to `Smoke-prod-Keychain-app` after the ASG incident.
- The `SSM_For_EC2` instance role is very broad: `AmazonSSMFullAccess`, `AmazonEC2FullAccess`, `SecretsManagerReadWrite`, `AmazonS3FullAccess`, `CloudWatchFullAccess`, `AWSLambda_FullAccess`, `AWSCodeDeployFullAccess`, and other policies were observed. Any shell access to an instance with this profile can likely obtain high-impact instance credentials from instance metadata.
- Running prod EC2 instances using `SSM_For_EC2` include `prod-kafka`, `Smoke-prod-Keychain-app`, `Odoo-production`, `Emqx-prod`, `wordpress-prod`, `wordpress-sensor-insure`, `prod-api-server`, and `prod-sso-api`.
- CloudTrail did not show `GetSecretValue` during the reviewed `chaitanya.sharma@appinventiv.com` SSM session window. Secret reads seen during the `aniket2@appinventiv.com` window were from the prod SSO instance role and matched normal application reads, not direct evidence from the Keychain session.
- `SSM-SessionManagerRunShell` was updated externally and verified on 2026-05-08. Default version `2` streams Session Manager output to CloudWatch Logs group `/aws/ssm/session-manager` with 90-day retention.

## Risks Or Constraints

- `Backup-user` was deleted after investigation, but it existed long enough to create an admin access key and should remain part of incident review.
- The incident sessions used federated console access with `MFAUsed: No` according to CloudTrail.
- Group-based direct IAM admin access is broad. `peresada@gmail.com` is confirmed expected; `tom.mcevoy@sensorglobal.com` has been reduced to `ReadOnly-users`.
- Expected admin user `Richard.Heywood@syncom.com.au` has no MFA devices configured.
- `cloudfromation-template-role` and `AWS-QuickSetup-StackSet-Local-ExecutionRole` still have direct `AdministratorAccess`.
- `lokendra.saini@appinventiv.com` was removed externally and verified by Codex with read-only checks.
- `MSP-user` still exists, although its direct admin policy and console login were removed.
- `security-hub-finding` has console login plus broad Security Hub, Inspector, and S3 permissions.
- The external audit role trusting account `043210536673` has no external ID condition and includes limited write permissions.
- IAM Access Analyzer is disabled, reducing continuous visibility into public or cross-account access.
- `sensor-prod` MySQL still has broad private network ingress from `10.0.0.0/16`; this makes any compromised prod VPC shell a database network path. Stale external/client CIDR rules have been removed.
- `openvpn-5SG` is public and can reach Odoo PostgreSQL.
- `SSM_For_EC2` has broad instance-role permissions, including Secrets Manager read/write and EC2 full access.
- SSM session command output logging is now configured to CloudWatch Logs; run a short non-sensitive test session to confirm stream creation.
- Codex did not perform any containment mutation because this repository's guardrails prohibit AWS changes.

## Cost Optimization Opportunities

None. This is a security and availability investigation.

## Recommended Next Steps

Immediate containment completed externally and verified:

1. `Backup-user` access key set inactive.
2. OneLogin SAML provider deleted.
3. `lokendra.saini@appinventiv.com` removed from IAM group `Admin`.
4. `AdministratorAccess` detached from `Backup-user`.
5. `Backup-user` access key deleted.
6. `Backup-user` deleted.
7. `AdministratorAccess` detached from `MSP-user`.
8. Console login profile deleted for `MSP-user`.
9. `AdministratorAccess` detached from `sensorglobal-admin`.
10. `sensorglobal-admin` deleted.
11. `lokendra.saini@appinventiv.com` deleted.
12. `tom.mcevoy@sensorglobal.com` removed from `Admin`, leaving `ReadOnly-users`.
13. Stale `sensor-prod` MySQL ingress rules removed from `DB-SG`: `42.108.28.155/32`, `20.0.0.0/22`, and UDP 443 from `20.0.0.0/22`.
14. SSM Session Manager logging enabled to CloudWatch Logs group `/aws/ssm/session-manager` with 90-day retention.

Recommended next steps:

1. Keep `peresada@gmail.com` as an expected operator, but review whether both `Admin` and `SuperAdmin` are still required.
2. Add MFA for `Richard.Heywood@syncom.com.au` before relying on the account for admin access.
3. Delete or fully disable `MSP-user` after confirming it is not required.
4. Review `security-hub-finding`; it has console login plus broad Security Hub, Inspector, and S3 permissions.
5. Review active long-lived keys for `peresada1@gmail.com`, `bitrise`, and `zabbix_monitoring`; rotate or remove where possible.
6. Split and narrow `SSM_For_EC2`; avoid using one broad EC2 role for SSM, Secrets Manager read/write, EC2 full access, S3 full access, Lambda full access, and CodeDeploy full access.
7. Restrict `sensor-prod` MySQL security group ingress away from broad `10.0.0.0/16`; prefer explicit application, SSO, EKS, and controlled jump-host security groups.
8. Validate or retire the public OpenVPN path; it can reach Odoo PostgreSQL.
9. Review or remove `cloudfromation-template-role` and `AWS-QuickSetup-StackSet-Local-ExecutionRole` if no longer required.
10. Review the external trust for `Audits-Automation-Scripts-Role-Sensor-Global`; add an external ID or remove the trust if obsolete.
11. Re-enable or recreate IAM Access Analyzer for the standalone account.
12. Add alerting for `UpdateAutoScalingGroup` setting production ASGs to zero and for IAM events such as `CreateUser`, `CreateAccessKey`, `AttachUserPolicy`, and `PutUserPolicy`.
13. Delete inactive access keys after ownership is confirmed.

Suggested read-only follow-up:

- Broaden CloudTrail search for all mutating events from Appinventiv SAML sessions and `Backup-user` across the incident window.
- Review OneLogin audit logs for login source, MFA policy, and assignment changes.
- Review whether `cloudfromation-template-role` and Quick Setup roles still need administrator permissions.

## Open Questions

- Was `Backup-user` an approved emergency/back-office user, or was it created outside process?
- Who currently controls OneLogin group membership for `sensorglobal-admin`?
- Was MFA disabled, bypassed, or simply not required for this OneLogin application?
- Are there approved break-glass users, and are they documented with expiry and monitoring?
- What is external AWS account `043210536673`, and is `Audits-Automation-Scripts-Role-Sensor-Global` still required?
