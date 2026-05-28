# Applied Changes

Record only approved changes that were actually applied.

### 2026-05-11 - Odoo outgoing mail server pointed to SES
- **Approval reference:** User obtained Odoo admin access, changed the existing Odoo outgoing mail server record in the UI, and asked Codex to verify.
- **Applied by:** User/operator; Codex ran read-only verification afterward.
- **Execution location:** external to Codex - Odoo production UI at `https://sensorglobal.com/web`.
- **Systems affected:** Odoo production database `sensorglobal_live`; Odoo outgoing mail server record `sendgrid smtp`; AWS SES in `ap-southeast-2`.
- **Summary:** Updated the active Odoo outgoing mail server named `sendgrid smtp` so compatibility with custom Odoo code is preserved while SMTP delivery uses SES instead of SendGrid. The verified live record now points to `email-smtp.ap-southeast-2.amazonaws.com` on port `587` with `starttls`.
- **Verification:** Codex verified read-only through approved SSM diagnostics on `i-0c5073e95aba77614` that `ir_mail_server` record `(11, 'sendgrid smtp')` now has `smtp_host=email-smtp.ap-southeast-2.amazonaws.com`, `smtp_port=587`, `smtp_encryption=starttls`, and `active=True`. SES account status remains healthy with `ProductionAccessEnabled=true`, `SendingEnabled=true`, `EnforcementStatus=HEALTHY`, quota `50000/day`, send rate `14/sec`, and `SentLast24Hours=4.0`. The historical `mail_mail` exception backlog remains unchanged at verification time: `cancel=9734`, `exception=1806`, `sent=3300`. User then sent an Odoo invitation email and confirmed it was received.
- **Notes:** Codex did not perform the Odoo mutation. The host-level Postfix relay container still needs a separate approved migration if flows use `postfix-static-relay`; earlier diagnostics showed it pointing to `[smtp.sendgrid.net]:587`. The primary Odoo UI outgoing-mail path is accepted based on successful invitation receipt.

### 2026-05-10 - Backend email provider cut over to SES
- **Approval reference:** User confirmed they changed the production backend email provider to SES, requested restart instructions, and then confirmed the corrected SSM restart commands were run.
- **Applied by:** User/operator; Codex ran read-only verification afterward.
- **Execution location:** external to Codex - AWS Secrets Manager update and SSM Run Command in account `747293622182`, region `ap-southeast-2`.
- **Systems affected:** AWS Secrets Manager secret `sensor-prod`; production backend EC2 instances `i-019a920e221d70159` and `i-0e1aaef0f8a051469`; PM2 application `Smoke_alarm`; ALB target group `SmokeAPI`.
- **Summary:** Updated production backend runtime configuration from dormant SES credentials to active SES provider configuration by setting `EMAIL_PROVIDER=ses`, then restarted the production backend PM2 application on both active API instances so the processes reload `sensor-prod`.
- **Verification:** Codex verified read-only that `sensor-prod` parses as valid JSON, `EMAIL_PROVIDER=ses`, `PROVIDER=SendGrid`, `SES_HOST=email-smtp.ap-southeast-2.amazonaws.com`, `SES_PORT=587`, and both `SES_USER` and `SES_PASSWORD` keys are present and non-empty without printing their values. `describe-secret` shows the current version changed at `2026-05-10T08:39:41+10:00`. Initial SSM command `4f163093-8620-4a5f-874c-bc2536f571cf` ran PM2 under the wrong home and reported no real process. Corrected SSM commands `4f3483c0-0a36-42a3-a8e7-6e147716689e` and `1024cea5-0b66-4285-9f8c-1e3ece09cfd6` both completed with `Status=Success` and `ResponseCode=0`. `SmokeAPI` target health remains `healthy` for both instances. SES account status remains healthy with `ProductionAccessEnabled=true`, `SendingEnabled=true`, `EnforcementStatus=HEALTHY`, quota `50000/day`, send rate `14/sec`, and `SentLast24Hours=0.0`.
- **Notes:** Post-cutover smoke verification completed: user triggered a reset-password email for test agency `59120`, confirmed the email was received, and confirmed the rendered template looked correct. Codex verified read-only afterward that SES `SentLast24Hours=1.0` and both `SmokeAPI` targets remained healthy. Rollback is `EMAIL_PROVIDER=sendgrid` plus the same PM2 restart on both production API instances.

### 2026-05-10 - SES SMTP credentials stored dormant in backend secret
- **Approval reference:** User created SES SMTP credentials, added them to `sensor-prod`, fixed the JSON syntax issue, and confirmed completion.
- **Applied by:** User/operator; Codex ran read-only verification afterward.
- **Execution location:** external to Codex - AWS Secrets Manager update in account `747293622182`, region `ap-southeast-2`.
- **Systems affected:** AWS Secrets Manager secret `sensor-prod`.
- **Summary:** Added dormant SES SMTP configuration to the production backend secret while keeping `EMAIL_PROVIDER=sendgrid`, so the running backend remains configured for SendGrid until a later controlled cutover.
- **Verification:** Codex verified read-only that `sensor-prod` parses as valid JSON, `PROVIDER=SendGrid`, `EMAIL_PROVIDER=sendgrid`, `SES_HOST=email-smtp.ap-southeast-2.amazonaws.com`, `SES_PORT=587`, and both `SES_USER` and `SES_PASSWORD` keys are present and non-empty without printing their values. `describe-secret` shows the current version changed at `2026-05-10T08:38:19+10:00`.
- **Notes:** No backend restart/deploy or provider cutover was performed. To cut over later, change `EMAIL_PROVIDER` to `ses` and restart/deploy backend during a controlled window. Rollback is `EMAIL_PROVIDER=sendgrid` plus backend restart/deploy.

### 2026-05-10 - SES production access granted
- **Approval reference:** User submitted the SES production access request externally and confirmed completion with "done".
- **Applied by:** User/operator; Codex ran read-only verification afterward.
- **Execution location:** external to Codex - AWS CLI or console in account `747293622182`, region `ap-southeast-2`.
- **Systems affected:** AWS SES account-level sending status in `ap-southeast-2`.
- **Summary:** SES production access was granted for transactional email from `sensorglobal.com`. This removes the SES sandbox limit for the account in `ap-southeast-2`; the backend still remains on SendGrid unless `EMAIL_PROVIDER=ses` and SES SMTP credentials/config are set later.
- **Verification:** Codex verified read-only that `sesv2 get-account --region ap-southeast-2` reports `ProductionAccessEnabled=true`, `SendingEnabled=true`, `EnforcementStatus=HEALTHY`, quota `Max24HourSend=50000`, send rate `MaxSendRate=14`, `MailType=TRANSACTIONAL`, `WebsiteURL=https://sensorglobal.com`, and review status `GRANTED` with case `177836478300309`. `sesv2 get-email-identity --email-identity sensorglobal.com --region ap-southeast-2` still reports `VerifiedForSendingStatus=true`, DKIM `Status=SUCCESS`, and custom MAIL FROM `MailFromDomainStatus=SUCCESS`.
- **Notes:** Next step is creating/storing SES SMTP credentials and doing a controlled backend config cutover. No backend deploy or email provider switch was performed by this change.

### 2026-05-10 - Failed api.sensorglobal.com SES identity removed
- **Approval reference:** User requested removal of the failed `api.sensorglobal.com` SES identity and then confirmed it was deleted.
- **Applied by:** User/operator; Codex ran read-only verification afterward.
- **Execution location:** external to Codex - AWS CLI or console in account `747293622182`, region `ap-southeast-2`.
- **Systems affected:** AWS SES identity `api.sensorglobal.com`.
- **Summary:** Removed the stale failed SES domain identity `api.sensorglobal.com`. This did not change the production API DNS record; `api.sensorglobal.com` remains a Route53 CNAME to the production API load balancer.
- **Verification:** Codex verified read-only that `sesv2 list-email-identities --region ap-southeast-2` now lists only `sensorglobal.com`. `sesv2 get-email-identity --email-identity sensorglobal.com` still reports `VerifiedForSendingStatus=true`, `VerificationStatus=SUCCESS`, DKIM `Status=SUCCESS`, and `MailFromDomainStatus=SUCCESS`.
- **Notes:** SES account production access is still not enabled: `ProductionAccessEnabled=false`, quota `200/day`, send rate `1/sec`.

### 2026-05-10 - SES domain identity provisioned for sensorglobal.com
- **Approval reference:** User agreed to use `sensorglobal.com` for SES, approved proceeding with Terraform, then confirmed the Terraform apply was completed.
- **Applied by:** User/operator; Codex prepared Terraform and ran read-only verification afterward.
- **Execution location:** external to Codex - Terraform account environment under `code/infra/terraform/environments/account`.
- **Systems affected:** AWS SES in account `747293622182`, region `ap-southeast-2`; Route53 hosted zone `Z03520551S3PY3J02L5ZN` / `sensorglobal.com`.
- **Summary:** Created SES domain identity `sensorglobal.com`, three SES Easy DKIM CNAME records under `_domainkey.sensorglobal.com`, and custom MAIL FROM domain `mail.sensorglobal.com` with MX and SPF records. The backend remains on SendGrid unless `EMAIL_PROVIDER=ses` is explicitly set later.
- **Verification:** Codex verified read-only that `sesv2 get-email-identity --email-identity sensorglobal.com` reports `VerifiedForSendingStatus=true`, `VerificationStatus=SUCCESS`, DKIM `Status=SUCCESS`, and `MailFromDomainStatus=SUCCESS` for `mail.sensorglobal.com`. Route53 contains SES DKIM CNAMEs for tokens `e6bihsejv2f27r3bnznqsrczlyjjdfgd`, `qh5cz3xa3qxa3zfra4iillrhoruyr6ja`, and `3r2dmwzllwrl5v7wqgk3qpmmi7dcxzrg`, plus `mail.sensorglobal.com` MX `10 feedback-smtp.ap-southeast-2.amazonses.com` and TXT `v=spf1 include:amazonses.com -all`.
- **Notes:** SES production access is still not enabled: `sesv2 get-account` reports `ProductionAccessEnabled=false`, quota `200/day`, and send rate `1/sec`. Next step is SES production access approval, followed by SMTP credential creation/storage and a separate controlled cutover.

### 2026-05-09 - Production backend deployment pipeline validation
- **Approval reference:** User requested a minimal production pipeline validation change, chose to make the code/docs change, then confirmed the validation branch was pushed and merged.
- **Applied by:** User/operator; Codex ran read-only verification afterward.
- **Execution location:** external to Codex - GitHub PR merge to `SensorGlobal/sensor-alarm-backend` `master`, followed by AWS CodePipeline/CodeBuild/CodeDeploy.
- **Systems affected:** GitHub repository `SensorGlobal/sensor-alarm-backend`; CodePipeline `Smoke-API`; CodeBuild `Smoke-API`; CodeDeploy application/deployment group `Smoke-API/Smoke-API`; Auto Scaling group `CodeDeploy_Smoke-API_d-4CJSQ2O16`; target group `SmokeAPI`.
- **Summary:** Merged docs-only validation PR #6020 from `docs/pipeline-validation-check` into `master`, producing source revision `f728a1905a85b86fdd00c15e75277218cdc93d51`. The production backend pipeline picked up the commit, built the artifact, deployed it through CodeDeploy blue/green, and shifted traffic to a new API Auto Scaling group.
- **Verification:** Codex verified read-only that CodePipeline `Smoke-API` Source, Build, and Deploy stages all succeeded. CodeBuild execution `Smoke-API:95abd387-e8ff-4e85-a5ce-df832eb3aa07` succeeded after `PRE_BUILD`, `BUILD`, and artifact upload. CodeDeploy deployment `d-4CJSQ2O16` succeeded at `2026-05-09T08:24:08+10:00` with `Succeeded=4`, `Failed=0`. Target group `SmokeAPI` now has two healthy targets, `i-048f8cfc816ae158b` and `i-07a2cbcdd473b6fac`, on port `4553`.
- **Notes:** Codex did not mutate AWS. Raw CodeBuild logs were not fetched because the legacy inline buildspec runs `cat .env`; status-only and structured AWS APIs were used to avoid exposing secrets. Follow-up recommendation: add a runtime build/version signal and remove `.env` printing from the CodeBuild buildspec in a controlled deployment change.

### 2026-05-08 - SSM Session Manager logging enabled
- **Approval reference:** User approved proceeding with the SSM session logging slice and confirmed the Session Manager document update plus default-version switch were run.
- **Applied by:** User/operator
- **Execution location:** external to Codex - AWS CLI with MFA-backed IAM user session
- **Systems affected:** SSM Session Manager preferences document `SSM-SessionManagerRunShell`; CloudWatch Logs log group `/aws/ssm/session-manager`
- **Summary:** Created CloudWatch Logs log group `/aws/ssm/session-manager`, set 90-day retention, updated `SSM-SessionManagerRunShell` to version `2` with `cloudWatchLogGroupName=/aws/ssm/session-manager` and `cloudWatchStreamingEnabled=true`, then made version `2` the default.
- **Verification:** Codex ran read-only verification. `ssm describe-document --name SSM-SessionManagerRunShell` reports `LatestVersion=2` and `DefaultVersion=2`. `ssm get-document --name SSM-SessionManagerRunShell` returns document version `2` with `cloudWatchLogGroupName` set to `/aws/ssm/session-manager`, `cloudWatchStreamingEnabled=true`, and no S3 destination. `logs describe-log-groups --log-group-name-prefix /aws/ssm/session-manager` shows the log group exists with `retentionInDays=90`.
- **Notes:** No log streams were expected until a new SSM session starts after the default-version switch. Run a short non-sensitive test session to confirm stream creation when convenient.

### 2026-05-08 - Production backend SMS globally disabled
- **Approval reference:** User requested a temporary full SMS disable, approved the documented `sensor-prod.SEND_SMS=0` plan, then confirmed the production API restart was completed.
- **Applied by:** User/operator; Codex ran read-only verification afterward.
- **Execution location:** external to Codex - AWS Secrets Manager update and production API PM2 restart
- **Systems affected:** Production Secrets Manager secret `sensor-prod`; production `sensor-alarm-backend` API PM2 processes in account `747293622182`, region `ap-southeast-2`
- **Summary:** Temporarily disabled all backend-driven SMS by changing `sensor-prod.SEND_SMS` from `1` to `0`, then restarted the production backend API processes so the running application would reload the updated secret. This is a global backend SMS pause and includes OTPs, alarm notifications, job SMS, tenant/landlord SMS, test alarm SMS, and queued SMS.
- **Verification:** Codex confirmed by read-only Secrets Manager check that `sensor-prod.SEND_SMS=0` and `SEND_SMS_ONLY_ON_ALLOWED_LIST=0`. Post-restart `scripts/health-check.sh --env prod` reported ALB healthy hosts `2`, ALB 5xx `0.0%`, API ASG `2/5/2` with `2` InService, SSO discovery `HTTP 200`, API invalid-login path `HTTP 401`, RDS CPU `5.6%`, RDS free memory `1508 MB`, RDS free storage `90.1 GB`, RDS connections `20`, and EC2 status `4/4`. Backend API deep health was skipped because `HEALTH_API_KEY` was not present in the local shell. SMS sends last 24h remained `41`, which is expected to decay after the approved pause window.
- **Notes:** Re-enable by restoring `sensor-prod.SEND_SMS=1` and restarting the production backend API processes again. See `docs/investigations/2026-05-08-prod-sms-global-disable.md` for scope, impact, and rollback details.

### 2026-05-08 - Stale production MySQL ingress rules removed
- **Approval reference:** User asked whether investigation tunnels would still work, then confirmed the stale database security group rules were removed.
- **Applied by:** User/operator
- **Execution location:** external to Codex - AWS CLI with MFA-backed IAM user session
- **Systems affected:** EC2 security group `sg-0aa1e92c7b3a5c58e` / `DB-SG`, production RDS MySQL network access
- **Summary:** Removed stale `DB-SG` ingress for MySQL TCP 3306 from `42.108.28.155/32` and `20.0.0.0/22`, and removed the stale UDP 443 rule from `20.0.0.0/22`. Kept the explicit application security group rules and the broad `10.0.0.0/16` prod VPC rule for a later controlled pass.
- **Verification:** Codex ran read-only verification. `ec2 describe-security-groups --group-ids sg-0aa1e92c7b3a5c58e` now shows only MySQL TCP 3306 from `sg-0830a4cf088cc54be` / Production API SG, `sg-0d544dd0240803824` / prod SSO API SG, `sg-043324995d224dca0` / `PROD_AWS_CLIENT_VPN`, and CIDR `10.0.0.0/16` / `prod-vpc-cidr`. The stale `42.108.28.155/32` and `20.0.0.0/22` CIDRs are absent, and there is no remaining UDP 443 ingress. Regional CloudTrail in `ap-southeast-2` shows `RevokeSecurityGroupIngress` for `42.108.28.155/32` at 2026-05-08 11:13:44 local time and for `20.0.0.0/22` TCP 3306 at 2026-05-08 11:13:54 local time, both from `peresada1@gmail.com` with `mfaAuthenticated=true`.
- **Notes:** Existing SSM/RDS investigation tunneling remains possible because `10.0.0.0/16` is still allowed. Before removing that broad VPC CIDR, add explicit security group access for every required DB client and a controlled investigation/jump-host path.

### 2026-05-08 - Lokendra removed and Tom admin access reduced
- **Approval reference:** User stated `lokendra.saini@appinventiv.com` must go, Tom should stay for now with reduced blast radius if possible, then confirmed "lokendra and tom done".
- **Applied by:** User/operator
- **Execution location:** external to Codex - AWS CLI with MFA-backed IAM user session
- **Systems affected:** IAM user `lokendra.saini@appinventiv.com`, IAM user `tom.mcevoy@sensorglobal.com`, IAM group `Admin`, IAM group `ReadOnly-users`
- **Summary:** Deleted IAM user `lokendra.saini@appinventiv.com`. Removed `tom.mcevoy@sensorglobal.com` from the `Admin` group while keeping him in `ReadOnly-users`.
- **Verification:** Codex ran read-only verification. `iam get-user --user-name lokendra.saini@appinventiv.com` returns `NoSuchEntity`, and `iam get-login-profile --user-name lokendra.saini@appinventiv.com` also returns `NoSuchEntity`. `iam list-groups-for-user --user-name tom.mcevoy@sensorglobal.com` returns only `ReadOnly-users`. `iam get-group --group-name Admin` now lists `peresada@gmail.com`, `kristyn.heywood@syncom.com.au`, `Richard.Heywood@syncom.com.au`, and `peresada1@gmail.com`; Tom and Lokendra are absent. CloudTrail shows `DeleteUser` for `lokendra.saini@appinventiv.com` at 2026-05-08 11:07:54 local time and `RemoveUserFromGroup` for `tom.mcevoy@sensorglobal.com` from `Admin` at 2026-05-08 11:08:12 local time, both from `peresada1@gmail.com` with `mfaAuthenticated=true`.
- **Notes:** Codex did not perform the IAM mutations. Remaining access-hardening items are database security group cleanup, OpenVPN containment, SSM role narrowing, SSM session logging, Richard MFA, and review of broad service-style users/roles.

### 2026-05-08 - Production SSO basemodels grantId index dropped again
- **Approval reference:** User requested production login recovery, Codex diagnosed the SSO token failure down to the MongoDB `basemodels.payload.grantId_1` unique index, and the user applied the index drop from the SSO instance shell.
- **Applied by:** User/operator
- **Execution location:** external to Codex - SSM shell on production SSO instance `i-01606ffe2138c0c8e`, using the runtime `sensor-prod-sso` MongoDB connection
- **Systems affected:** MongoDB Atlas cluster `sensor-prod`, database `sensorproddb`, collection `basemodels`; production SSO login flow
- **Summary:** Dropped the `payload.grantId_1` unique index from `sensorproddb.basemodels`. The index had reappeared after the earlier May 1 recovery because the deployed SSO schema still declares that index as unique. It is incompatible with `oidc-provider` password-grant token issuance because `AccessToken` and `RefreshToken` records can share the same `grantId`.
- **Verification:** User-provided output showed `payload.grantId_1` present before the change and absent afterward. The local SSO token probe then returned `status=200` with non-zero token lengths: `access_token_len=43`, `id_token_len=1130`, `refresh_token_len=43`, for `userId=59117`, `userType=8`.
- **Notes:** Codex did not perform the MongoDB mutation. This is an immediate recovery only; the durable fix is to remove `unique: true` from `code/sso-provider/src/db/mongodb/models/BaseModel.ts` and deploy SSO so future starts do not recreate the bad index.

### 2026-05-08 - MSP-user and orphaned sensorglobal-admin access removed
- **Approval reference:** User completed the follow-up access containment commands from `docs/runbooks/12-aws-access-containment-follow-up.md` and asked Codex to verify.
- **Applied by:** User/operator
- **Execution location:** external to Codex - AWS CLI with MFA-backed IAM user session
- **Systems affected:** IAM user `MSP-user`, IAM role `sensorglobal-admin`, policy attachment `AdministratorAccess`
- **Summary:** Detached direct `AdministratorAccess` from `MSP-user`, deleted the `MSP-user` console login profile, detached `AdministratorAccess` from the orphaned OneLogin admin role `sensorglobal-admin`, and deleted the `sensorglobal-admin` role.
- **Verification:** Codex ran read-only verification. `iam list-attached-user-policies --user-name MSP-user` returns an empty list. `iam get-login-profile --user-name MSP-user` returns `NoSuchEntity`. `iam get-role --role-name sensorglobal-admin` returns `NoSuchEntity`. `iam list-entities-for-policy --policy-arn arn:aws:iam::aws:policy/AdministratorAccess` now shows no direct users, and only roles `cloudfromation-template-role` and `AWS-QuickSetup-StackSet-Local-ExecutionRole` plus groups `Admin` and `SuperAdmin`. CloudTrail shows `DetachUserPolicy` for `MSP-user` at 2026-05-08 08:22:54 local time, `DeleteLoginProfile` at 08:25:27, `DetachRolePolicy` for `sensorglobal-admin` at 08:29:22, and `DeleteRole` at 08:29:33, all from `peresada1@gmail.com` with `mfaAuthenticated=true`.
- **Notes:** `MSP-user` still exists but currently has no direct admin policy, no login profile, and no access keys. Remaining broad access is documented in `docs/investigations/2026-05-08-aws-admin-access-and-asg-scale-down.md`.

### 2026-05-08 - Appinventiv OneLogin and Backup-user AWS access contained
- **Approval reference:** User asked how to disable Appinventiv/OneLogin AWS access, then confirmed the containment commands were completed and requested verification.
- **Applied by:** User/operator
- **Execution location:** external to Codex - AWS CLI with MFA-backed IAM user session
- **Systems affected:** IAM SAML provider `arn:aws:iam::747293622182:saml-provider/onelogin`, IAM user `Backup-user`, access key `AKIA237RLX6TDZFDABN2`, policy attachment `AdministratorAccess`, IAM group `Admin`
- **Summary:** Set the `Backup-user` access key inactive, deleted the OneLogin/Appinventiv SAML provider used to assume `sensorglobal-admin`, removed `lokendra.saini@appinventiv.com` from IAM group `Admin`, detached `AdministratorAccess` from `Backup-user`, deleted the `Backup-user` access key, and deleted the `Backup-user` IAM user.
- **Verification:** Codex ran read-only verification. `iam list-saml-providers` no longer shows `onelogin`; only `DEV-VPN` remains. `iam get-user --user-name Backup-user` returns `NoSuchEntity`. CloudTrail shows `UpdateAccessKey` setting `AKIA237RLX6TDZFDABN2` inactive at 2026-05-08 08:00:34 local time, `DeleteSAMLProvider` at 08:00:46, `RemoveUserFromGroup` for `lokendra.saini@appinventiv.com` from `Admin` at 08:00:57, `DetachUserPolicy` at 08:01:08, `DeleteAccessKey` at 08:01:18, and `DeleteUser` at 08:01:27, all from `peresada1@gmail.com` with `mfaAuthenticated=true`.
- **Notes:** This was followed by removal of the orphaned `sensorglobal-admin` role and `MSP-user` direct admin access. Remaining AWS entry points and follow-up recommendations are documented in `docs/investigations/2026-05-08-aws-admin-access-and-asg-scale-down.md`.

### 2026-05-08 - Production API and SSO ASG capacity restored
- **Approval reference:** User requested production recovery after Codex identified zero-target ALB `503` impact, then confirmed the recovery commands were applied.
- **Applied by:** User/operator
- **Execution location:** external to Codex - AWS CLI or AWS console with production AWS access
- **Systems affected:** Auto Scaling groups `CodeDeploy_Smoke-API_d-NNRF41DGD` and `CodeDeploy_sso-api-dg_d-QKLSZP50D`, target groups `SmokeAPI` and `sso-api-tg`, public endpoints `api.sensorglobal.com` and `auth.sensorglobal.com`
- **Summary:** Restored production API ASG capacity to `min=2`, `max=5`, `desired=2` and production SSO ASG capacity to `min=1`, `max=2`, `desired=1` after a prior console/API update had set both groups to `min=0`, `max=0`, `desired=0`.
- **Verification:** Codex ran read-only verification. API ASG shows two healthy `InService` instances, SSO ASG shows one healthy `InService` instance, API target group `SmokeAPI` has two healthy targets, SSO target group `sso-api-tg` has one healthy target, SSO discovery returns `HTTP 200`, and `scripts/health-check.sh --env prod --profile sensorsyn-mfa` reports `ALB healthy hosts 2`, `ALB 5xx error rate 0.0%`, `EC2 instances ok/total 4/4`, and no failing infrastructure metrics. Backend API-specific health remained skipped because `HEALTH_API_KEY_PROD` was not present in the local shell.
- **Notes:** CloudTrail identified the preceding scale-to-zero change at 2026-05-07 21:11 Australia/Sydney from an assumed `sensorglobal-admin` session for `chaitanya.sharma@appinventiv.com`. Codex did not run the production recovery mutation.

### 2026-05-07 - Safer Ops production EKS foundation provisioned
- **Approval reference:** User approved prod Terraform edits to provision the Safer Ops EKS cluster and later confirmed the cluster was provisioned and connectable.
- **Applied by:** User
- **Execution location:** external to Codex - Terraform apply with MFA-backed AWS access
- **Systems affected:** EKS, EC2 managed node group, IAM roles, KMS, CloudWatch Logs, and VPC subnet tags in account `747293622182`, region `ap-southeast-2`
- **Summary:** Provisioned EKS cluster `safer-ops-prod` with Kubernetes `1.35`, two `t3.medium` AL2023 managed nodes in private subnets, EKS managed add-ons `vpc-cni`, `kube-proxy`, and `coredns`, KMS secret encryption key `alias/safer-ops-eks`, control-plane log group `/aws/eks/safer-ops-prod/cluster`, EKS cluster role `SaferOpsEksClusterRole`, and node role `SaferOpsEksNodeRole`. Added Kubernetes discovery tags to the existing public and private app subnets for future load balancer integration.
- **Verification:** User confirmed the cluster was provisioned and connectable. Codex ran read-only verification: `kubectl get nodes -o wide` showed two nodes `Ready` on Amazon Linux 2023 with Kubernetes `v1.35.4-eks-40737a8`; `kubectl get pods -A` showed `aws-node`, `coredns`, and `kube-proxy` pods all `Running`.
- **Notes:** Codex prepared and validated the Terraform edits and plans but did not run the remote apply. The cluster API endpoint is `https://9425A24873F732096BDDE6819365937D.gr7.ap-southeast-2.eks.amazonaws.com`.

### 2026-05-07 - Safer Ops EKS IAM OIDC provider registered
- **Approval reference:** User approved prod Terraform edits for the EKS OIDC provider, then confirmed the Terraform apply output with `safer_ops_eks_oidc_provider_arn`.
- **Applied by:** User
- **Execution location:** external to Codex - Terraform apply with MFA-backed AWS access
- **Systems affected:** IAM OIDC provider in account `747293622182`, region `ap-southeast-2`
- **Summary:** Registered the EKS OIDC issuer as IAM OIDC provider `arn:aws:iam::747293622182:oidc-provider/oidc.eks.ap-southeast-2.amazonaws.com/id/9425A24873F732096BDDE6819365937D` with client id `sts.amazonaws.com`, enabling IRSA for future Kubernetes service account roles such as AWS Load Balancer Controller and External DNS.
- **Verification:** User-provided Terraform output confirmed `safer_ops_eks_oidc_provider_arn`, `safer_ops_eks_oidc_issuer`, cluster name, endpoint, node role ARN, and cluster security group ID. Codex verified before apply that the plan was `1 to add, 0 to change, 0 to destroy` and that the only planned AWS resource was `aws_iam_openid_connect_provider.safer_ops_eks`.
- **Notes:** Codex prepared and validated the Terraform edits and plan but did not run the remote apply. The Terraform `tls` provider is used to derive the OIDC thumbprint from the live issuer certificate.

### 2026-05-06 - Safer Ops ECR image publishing bootstrap
- **Approval reference:** User requested AWS CLI bootstrap script for Safer Ops image publishing and confirmed the script completed with image publisher role ARN `arn:aws:iam::747293622182:role/SaferOpsGitHubImagePublisherRole`.
- **Applied by:** User
- **Execution location:** external to Codex - AWS CLI with MFA profile
- **Systems affected:** ECR and IAM in account `747293622182`, region `ap-southeast-2`
- **Summary:** Created ECR repositories `safer-ops-api` and `safer-ops-web`, GitHub Actions OIDC provider `token.actions.githubusercontent.com`, IAM role `SaferOpsGitHubImagePublisherRole`, IAM policy `SaferOpsEcrImagePublisher`, and attached the policy to the role. The role trust policy allows `repo:SaferHomesAu/safer-ops:environment:production` to assume the role through GitHub OIDC.
- **Verification:** Codex ran read-only verification. Both ECR repositories exist; `safer-ops-api` URI is `747293622182.dkr.ecr.ap-southeast-2.amazonaws.com/safer-ops-api`, and `safer-ops-web` URI is `747293622182.dkr.ecr.ap-southeast-2.amazonaws.com/safer-ops-web`. The GitHub OIDC provider exists with `sts.amazonaws.com` as a client id. The publisher role exists with the expected production-environment trust policy and has `SaferOpsEcrImagePublisher` attached. GitHub Actions successfully published API and web images tagged `main` and `9515e143032724562233d21b0c9219771f71d004` to ECR.
- **Notes:** `safer-ops-api` was first created manually in the AWS console and currently has scan-on-push disabled; `safer-ops-web` was created by the script with scan-on-push enabled. Codex did not perform the AWS mutations.

### 2026-05-05 - Safer Ops production SSO client registered
- **Approval reference:** User approved creating `safer_ops_client` in the production SSO Mongo `clients` collection with redirect URI `https://safer-ops.sensorglobal.com/auth/callback`, then confirmed the Atlas insert was completed.
- **Applied by:** User
- **Execution location:** external to Codex - MongoDB Atlas UI
- **Systems affected:** Production SSO MongoDB Atlas collection `sensorproddb.clients`
- **Summary:** Added the Safer Ops OIDC web client `safer_ops_client` for production SSO login. The client allows `authorization_code` and `refresh_token` grants, uses scope `openid email profile offline_access api:read`, and has a registered callback at `https://safer-ops.sensorglobal.com/auth/callback`.
- **Verification:** Codex ran read-only verification against the production SSO MongoDB collection using the existing `sensor-prod-sso` secret. The document exists, `client_secret` is present, and the verified fields match the approved client id, redirect URI, grants, scope, application type, and `__v=0`.
- **Notes:** Codex did not perform the production MongoDB insert. Keep the generated client secret only in deployment secret storage and local secure `.env` files.

### 2026-05-01 - MongoDB custody KMS key and S3 bucket
- **Approval reference:** User requested implementation and executed the AWS commands during the 2026-05-01 MongoDB custody migration walkthrough.
- **Applied by:** User
- **Execution location:** external to Codex - AWS CLI
- **Systems affected:** AWS KMS and S3 in account `747293622182`, region `ap-southeast-2`
- **Summary:** Created customer-managed KMS key `191ab3a4-a352-4d12-b057-610adae73a0c` for MongoDB custody dump encryption, created alias `alias/sensor-prod-mongo-custody`, and configured S3 bucket `sensor-prod-mongo-custody-747293622182` with public access blocked, versioning enabled, and default SSE-KMS encryption using the custody alias.
- **Verification:** User-provided AWS CLI output confirmed KMS key state `Enabled`, public access block all `true`, bucket versioning `Enabled`, and bucket encryption `SSEAlgorithm=aws:kms` with `KMSMasterKeyID=alias/sensor-prod-mongo-custody`.
- **Notes:** No MongoDB dump or restore was performed in this step.

### 2026-05-01 - MongoDB migration runner EC2 instance
- **Approval reference:** User requested implementation and executed the AWS commands during the 2026-05-01 MongoDB custody migration walkthrough.
- **Applied by:** User
- **Execution location:** external to Codex - AWS CLI
- **Systems affected:** EC2, IAM instance profile, and security group in account `747293622182`, region `ap-southeast-2`
- **Summary:** Created IAM role/profile `MongoMigrationRunnerRole` / `MongoMigrationRunnerProfile`, security group `sg-0dd980e523ba78d23`, and temporary migration runner EC2 instance `i-0c179962eb08ed4c6` for SSM-only MongoDB dump/restore work.
- **Verification:** User-provided AWS CLI output confirmed SSM registration for `i-0c179962eb08ed4c6` with `PingStatus=Online`, platform `Amazon Linux`, SSM agent `3.3.4108.0`. Earlier read-only checks confirmed the security group has no ingress and all outbound egress. User fixed a MongoDB repo URL typo, installed `mongodump`/`mongorestore` `100.16.0` and `mongosh` `2.8.2`, then uploaded `dumps/test/custody-test.txt` to the custody bucket with `ServerSideEncryption=aws:kms` and KMS key `191ab3a4-a352-4d12-b057-610adae73a0c`.
- **Notes:** Instance is temporary and should be terminated after dump/restore validation is complete. No production MongoDB dump or restore was performed in this step.

### 2026-05-01 - MongoDB custody dump uploaded to S3
- **Approval reference:** User requested implementation and executed the upload during the 2026-05-01 MongoDB custody migration walkthrough.
- **Applied by:** User
- **Execution location:** external to Codex - AWS CLI from migration runner
- **Systems affected:** S3 bucket `sensor-prod-mongo-custody-747293622182` in account `747293622182`, region `ap-southeast-2`
- **Summary:** Uploaded the completed production `mongodump --archive --gzip` file and its `.sha256` checksum into the custody bucket under `dumps/final/`.
- **Verification:** User confirmed the upload completed in chat. Object-level verification with `aws s3 ls` and `aws s3api head-object` should still be retained if available, especially to confirm SSE-KMS encryption, KMS key, size, version ID, and final object key.
- **Notes:** This is a custody copy only. It does not change production application configuration or production traffic.

### 2026-05-01 - MongoDB custody dump restored to JV Atlas cluster
- **Approval reference:** User requested implementation and executed the restore during the 2026-05-01 MongoDB custody migration walkthrough.
- **Applied by:** User
- **Execution location:** external to Codex - MongoDB Database Tools from migration runner
- **Systems affected:** MongoDB Atlas cluster `sensor-prod` in organization `Sensor Global Pty Ltd`, project `Sensor Production`
- **Summary:** Restored the production `mongodump --archive --gzip` custody dump into the new JV-controlled Atlas cluster. The first restore attempt failed due Atlas write-blocking from target disk exhaustion; after target storage was increased, the restore was rerun successfully.
- **Verification:** User-provided restore output ended with `73138048 document(s) restored successfully. 0 document(s) failed to restore.` The largest restored collection `sensorproddb.tbl_sim_event_logs` completed with `68334311 documents, 0 failures`. Post-restore target validation showed `16` collections, `1` view, `73137634` objects, `100` indexes, and a successful temporary write/drop test.
- **Notes:** Application cutover has not yet been performed. Production secrets and application traffic still require separate controlled cutover and rollback gates.

### 2026-05-01 - Production MongoDB application cutover to JV Atlas
- **Approval reference:** User requested "proceed" after successful restore validation and executed the approved Secrets Manager updates and PM2 restarts during the 2026-05-01 MongoDB custody migration walkthrough.
- **Applied by:** User; Codex ran read-only health and log verification afterward.
- **Execution location:** external to Codex - AWS CLI/SSM and Secrets Manager
- **Systems affected:** Production Secrets Manager secrets `sensor-prod` and `sensor-prod-sso`; production API and SSO PM2 processes; MongoDB Atlas target cluster `sensor-prod` in organization `Sensor Global Pty Ltd`
- **Summary:** Updated `sensor-prod.MONGO_DB_URL` and `sensor-prod-sso.MONGODB_URI` to the new JV-controlled Atlas cluster, then restarted root-owned PM2 processes for the production API and SSO instances so they would reload Secrets Manager values.
- **Verification:** Post-cutover `scripts/health-check.sh --env prod` passed and saved `tmp/baseline-after-mongo-cutover-20260501-133515.json`: ALB healthy hosts `2`, ALB 5xx `0.0%`, backend API `HTTP 200`, Redis connected, stalled hubs `5`, invalid invoices `1`, and SMS sends last 24h `53`. CloudWatch Logs showed recent `mongoose.connection.readyState >>> 1` entries after the cutover.
- **Notes:** `MONGO_DB_ARCHIVE_URL` was intentionally left unchanged for the initial cutover, but post-restart logs showed `MONGO_DB_ARCHIVED` connection errors against the old Atlas Online Archive endpoint. Treat archive recreation/retargeting as a follow-up before relying on historical archived data. Rotate/delete any temporary Atlas migration user whose URI appeared in terminal output.

### 2026-05-01 - Production MongoDB URI encoding fix
- **Approval reference:** User reported fresh incognito login failure after cutover, then updated the Atlas DB user passwords/URIs and restarted the affected services.
- **Applied by:** User; Codex ran read-only verification afterward.
- **Execution location:** external to Codex - AWS CLI/SSM and Secrets Manager
- **Systems affected:** Production Secrets Manager secrets `sensor-prod` and `sensor-prod-sso`; production API and SSO PM2 processes
- **Summary:** Corrected malformed MongoDB Atlas connection strings that contained unescaped characters in URI userinfo, then restarted root-owned PM2 processes for the production API and SSO instances.
- **Verification:** Restart command `e33de6af-b10a-4b19-b152-95a3a82d9856` succeeded on the three API instances and the SSO instance. Redacted URI diagnostics showed both `sensor-prod.MONGO_DB_URL` and `sensor-prod-sso.MONGODB_URI` point to `sensor-prod.yujwyg.mongodb.net` with no unsafe raw characters in userinfo. CloudWatch Logs showed fresh `Connected to MONGO_DB in bootstrap` events and no fresh `MongoParseError` in the checked window. `scripts/health-check.sh --env prod` saved `tmp/baseline-after-mongo-uri-fix-20260501-141353.json`; infrastructure checks passed, while backend API-specific checks were skipped because `HEALTH_API_KEY` was not present in the local shell.
- **Notes:** Confirm fresh browser login/logout manually. The Atlas Online Archive follow-up remains open because `MONGO_DB_ARCHIVE_URL` was not retargeted in this fix.

### 2026-05-01 - Production MongoDB database path fix
- **Approval reference:** User reported `client authentication failed` on fresh incognito login after the URI encoding fix; Codex diagnosed that both Mongo URIs were missing the `/sensorproddb` database path.
- **Applied by:** User; Codex ran read-only verification afterward.
- **Execution location:** external to Codex - AWS CLI/SSM and Secrets Manager
- **Systems affected:** Production Secrets Manager secrets `sensor-prod` and `sensor-prod-sso`; production API and SSO PM2 processes
- **Summary:** Updated `sensor-prod.MONGO_DB_URL` and `sensor-prod-sso.MONGODB_URI` so both connection strings target `/sensorproddb` on the new JV Atlas host, then restarted the three production API PM2 processes and the production SSO PM2 process.
- **Verification:** Restart command `c824f9bc-ba25-461c-82f0-3c39fee15196` succeeded on the three API instances and the SSO instance. Redacted URI diagnostics showed both `MONGO_DB_URL` and `MONGODB_URI` target host `sensor-prod.yujwyg.mongodb.net` with path `/sensorproddb`. CloudWatch Logs showed fresh `Connected to MONGO_DB in bootstrap` events, no fresh `MongoParseError`, no SSO `client authentication failed` entries, and no fresh SSO error entries in the checked window.
- **Notes:** Confirm fresh browser login/logout manually. The Atlas Online Archive follow-up remains open because `MONGO_DB_ARCHIVE_URL` was not retargeted in this fix.

### 2026-05-01 - Production SSO basemodels index compatibility fix
- **Approval reference:** User reported `400 VALIDATION_ERROR` / `oops! something went wrong` on login after the database path fix; Codex diagnosed an extra unique `payload.grantId_1` index on the restored JV Atlas `sensorproddb.basemodels` collection that was absent from the old Atlas cluster.
- **Applied by:** User; Codex ran read-only diagnosis afterward.
- **Execution location:** external to Codex - MongoDB Atlas / `mongosh`
- **Systems affected:** MongoDB Atlas cluster `sensor-prod` in organization `Sensor Global Pty Ltd`, collection `sensorproddb.basemodels`; production SSO login flow
- **Summary:** Dropped the extra `payload.grantId_1` unique index from `sensorproddb.basemodels`. The index was incompatible with the SSO `oidc-provider` storage pattern because successful login writes multiple token/session records that can share a grant id.
- **Verification:** Direct SSO password grant for the real test account previously returned `500 server_error` before the index fix, while bad-password/fake-user paths returned expected credential errors. After the index was dropped, the user confirmed browser login succeeded.
- **Notes:** No service restart was expected for this index-only fix. Continue manual login/logout checks and monitor SSO/API logs during the soak window.

## Entry Template

### YYYY-MM-DD - Short change title
- Approval reference:
- Applied by:
- Execution location: external to Codex | local repo only
- Systems affected:
- Summary:
- Verification:
- Notes:

---

### 2026-04-28 — AWS account detached from Ingram Micro parent organization
- **Approval reference:** User confirmed in chat: "ok, got approval. created additional account. did detach"
- **Applied by:** User / JV administrator
- **Execution location:** external to Codex — AWS Organizations member-account detach executed outside Codex
- **Systems affected:** AWS account `747293622182`; AWS Organizations membership and billing/governance boundary
- **Summary:** Account `747293622182` was detached from the Ingram Micro parent AWS Organization and now operates as a standalone AWS account under JV control. Workloads remained in the same AWS account; this was not a workload migration.
- **Verification:** `aws organizations describe-organization` now returns `AWSOrganizationsNotInUseException: Your account is not a member of an organization.` `aws sts get-caller-identity` still confirms account `747293622182`. Post-detach health check saved to `tmp/baseline-after-account-detach-20260428-144824.json` with ALB healthy hosts `3`, ALB 5xx `0.0%`, RDS CPU `5.5%`, RDS free storage `90.0 GB`, RDS connections `15`, EC2 status `7/7`, and SMS sends last 24h `107`. Route 53 Domains list still shows 14 domains visible with AutoRenew enabled.
- **Notes:** Root user ownership remains unrecovered and should be handled as a follow-up risk item. User later confirmed a new payment method was added in the AWS console on 2026-04-28. Billing alternate contact is still missing; operations and security alternate contacts are configured. Support API still reports `SubscriptionRequiredException`, so support plan verification remains pending in the AWS console if Business/Premium support is desired.

---

### 2026-04-11 — CloudWatch log retention set on non-prod environments
- **Approval reference:** Cost optimisation session, user approved retention automation
- **Applied by:** Copilot (scripts/set-log-retention.sh)
- **Execution location:** local repo — script ran against AWS API
- **Systems affected:** All CloudWatch log groups in dev, qa, sandbox
- **Summary:** Set 30-day retention on all log groups that had no retention policy (previously retained forever). Logged group names + previous settings before changing.
- **Verification:** `aws logs describe-log-groups` confirmed retention set; no application errors observed
- **Notes:** Script supports `--dry-run`; prod log groups were intentionally skipped

---

### 2026-04-12 — Terraform IaC baseline: all 5 environments imported
- **Approval reference:** User approved iterative Terraformer import across session turns
- **Applied by:** Copilot (terraform import, terraform state mv)
- **Execution location:** local repo — `code/infra/terraform/environments/{account,prod,dev,qa,sandbox}/`
- **Systems affected:** Terraform state only — no AWS resources were modified
- **Summary:** All existing infrastructure (VPCs, subnets, SGs, EC2, RDS, ALB, ElastiCache, EIPs, IAM roles, S3 buckets, Route53, ACM certs, CloudWatch) imported into Terraform state across all 5 env modules. State stored in S3 backend `sensorsyn-terraform-state-747293622182`. All 5 environments verified at `No changes` after import.
- **Verification:** `terraform plan` across all 5 envs shows `No changes. Your infrastructure matches the configuration.`
- **Notes:** Import runbook at `code/infra/terraform/docs/import-runbook.md`; account-level resources (Route53, shared ACM cert, IAM SSM role) isolated in `account/` module

---

### 2026-04-14 — Terraform workflow conventions added to copilot-instructions.md
- **Approval reference:** User approved "apply" of session-learned improvements
- **Applied by:** Copilot (edit to .github/copilot-instructions.md)
- **Execution location:** local repo only
- **Systems affected:** `.github/copilot-instructions.md`
- **Summary:** Added "Terraform workflow conventions" section with 4 rules learned from session: (1) execution posture — run plans/applies directly; (2) import safety — check all module states first, account-level resources only in `account/`; (3) destroy guard — any destroy in plan = stop and diagnose; (4) ALB cert rotation — purge expired SNI entries after attaching new cert.
- **Verification:** Section present in file; no AWS changes
- **Notes:** Rules prevent repeat of prod plan destroy incident and duplicate ACM/IAM ownership issue discovered during import

---

### 2026-04-16 — Locals extraction: hardcoded AWS IDs consolidated into locals.tf per environment
- **Approval reference:** User approved locals extraction as part of migration baseline work
- **Applied by:** Copilot (scripts/locals-extract.py)
- **Execution location:** local repo — script ran `terraform show -json` to pull state then rewrote .tf files
- **Systems affected:** `code/infra/terraform/environments/{dev,qa,sandbox,prod}/` — .tf files only, no AWS changes
- **Summary:** All hardcoded AWS resource IDs (vpc-xxx, sg-xxx, subnet-xxx, ami-xxx, eipalloc-xxx, eni-xxx, igw-xxx, nat-xxx, rtb-xxx, instance IDs) extracted from inline .tf files into a `locals.tf` per environment. Inline references replaced with `local.<name>` variables. ID counts: dev 48, qa 39, sandbox 49, prod 61.
- **Verification:** `terraform plan` on all 4 envs (account env has no IDs) shows `No changes` post-extraction.
- **Notes:** For migration to a new account, only `locals.tf` per env needs updating — all resource body code is now ID-free. `scripts/locals-extract.py` can be re-run to refresh if new IDs are added.

---

### 2026-04-16 — CloudWatch alarms and Secrets Manager metadata imported into Terraform
- **Approval reference:** User approved automated import of missing alarms/secrets and asked to capture UAT
- **Applied by:** Copilot (`scripts/generate-missing-imports.py`, `terraform import`)
- **Execution location:** local repo — `code/infra/terraform/environments/{dev,qa,sandbox,prod}/`
- **Systems affected:** Terraform state + generated Terraform files only; no AWS resources were modified
- **Summary:** Added generated Terraform config + import blocks for env-scoped CloudWatch alarms and Secrets Manager secrets, then imported them directly into state. Imported counts: dev `42 alarms + 2 secrets`, qa `48 + 2`, sandbox `25 + 2`, prod `84 + 2`.
- **Verification:** `terraform plan` shows `No changes` in dev, qa, and sandbox after import. Prod alarm/secret import completed, but the final full prod plan now shows one unrelated ASG size drift on `aws_autoscaling_group.prod_api_server` (not caused by the alarm/secret imports).
- **Notes:** `scripts/generate-missing-imports.py` classifies UAT/account/local resources but only writes to existing env dirs by default. UAT inventory was captured separately in `docs/investigations/2026-04-16-uat-environment-discovery.md`.

---

### 2026-04-17 — UAT environment scaffolded + stopped variable wired into non-prod envs
- **Approval reference:** User approved UAT keep-and-scaffold + stopped variable for dev/qa/sandbox
- **Applied by:** Copilot (file edits in local repo only)
- **Execution location:** local repo — `code/infra/terraform/environments/{dev,qa,sandbox,uat}/`
- **Systems affected:** Terraform config only — no AWS resources were modified
- **Summary:**
  1. Added `var.stopped` (bool, default false) to `variables.tf` for dev, qa, sandbox, and uat.
  2. Wired `var.stopped` into `desired_capacity` and `min_size` on all 6 non-prod ASGs (dev_sso, dev_api, qa_sso, qa_api, sandbox_sso_asg, sandbox_api_asg).
  3. Created `stopped.tfvars` per env with usage instructions (including RDS/ElastiCache note and `stop-nonprod.sh` cross-reference).
  4. Scaffolded `environments/uat/` with `backend.tf`, `main.tf`, `variables.tf`, `locals.tf` (ID placeholders from discovery doc), and `stopped.tfvars`. Backend initialized and pointing to `nonprod/uat/terraform.tfstate`.
- **Verification:**
  - `terraform validate` passes for dev, qa, sandbox.
  - `terraform plan` (stopped=false default): `No changes` for dev, qa, sandbox, uat.
  - `terraform plan -var-file=stopped.tfvars` for dev: shows `desired_capacity 2→0, min_size 2→0` on both ASGs — `Plan: 0 to add, 2 to change, 0 to destroy`. No destroy operations.
- **Notes:** RDS/ElastiCache stop requires `scripts/stop-nonprod.sh` — Terraform has no native "stop" for these services. UAT `locals.tf` has placeholder comments for IDs pending `terraform import` of UAT resources.

---

### 2026-04-17 — UAT Terraform import completed (base infra + alarms + secrets)
- **Approval reference:** User approved UAT resource import (continuation of IaC baseline work)
- **Applied by:** Copilot (`terraform import`, `scripts/generate-missing-imports.py`)
- **Execution location:** local repo — `code/infra/terraform/environments/uat/`
- **Systems affected:** Terraform state only — no AWS resources were modified
- **Summary:**
  1. Wrote all UAT HCL files from verified AWS inventory: `networking.tf` (VPC, IGW, NAT, 6 subnets, 3 route tables, 16 SGs), `compute.tf` (3 EIPs, 4 EC2, 2 LTs, 2 ASGs), `data.tf` (RDS, ElastiCache, subnet groups), `loadbalancer.tf` (2 ALBs, 2 TGs, 4 listeners), and `locals.tf` (central ID registry).
  2. Imported all ~63 base resources across 7 phases. Fixed SG `description` field to prevent forced replacement; fixed route table associations import format (must use `subnet_id/route_table_id` not `rtbassoc-xxx`).
  3. Reconciled HCL to match live AWS state: fixed ALB `idle_timeout`, `access_logs` bucket names, TG `healthy_threshold`/`interval`/`matcher`, RDS `backup_window`/`maintenance_window`/`parameter_group_name`, ElastiCache `maintenance_window`/`snapshot_window`, ASG `protect_from_scale_in`, VPC peering routes (`192.168.248.0/21` via `pcx-0378fb73738ca8785`), DB subnet group description.
  4. Ran `scripts/generate-missing-imports.py --env uat --write --import` to import 20 CloudWatch alarms and 2 Secrets Manager secrets.
  5. Removed `uat` from `UNSUPPORTED_OUTPUT_ENVS` in the script now that the env dir exists.
- **Verification:** `terraform plan` shows `Plan: 0 to add, 46 to change, 0 to destroy`. The 46 in-place changes are all safe: `default_tags` additions (Environment/ManagedBy/Repo), default attribute population, Name tag cleanups, and one legitimate port correction (433→443 typo on the smoke-uat-api HTTP redirect listener).
- **Notes:** Total UAT state: 85 resources (63 base + 20 alarms + 2 secrets). Cost-optim TODOs remain in `data.tf`: UAT RDS multi_az=true→false and gp2→gp3.

---

### 2026-04-29 — Terraform post-decommission drift reconciled for prod, sandbox, dev, and QA
- **Approval reference:** User requested "prod autoscaling, then sandbox and last dev, qa" and approved proceeding in-session
- **Applied by:** Codex
- **Execution location:** local repo — `code/infra/terraform/environments/{prod,sandbox,dev,qa}/`
- **Systems affected:** Terraform config only — no AWS resources were modified and no Terraform apply/state mutation was run
- **Summary:**
  1. Reconciled prod API ASG config to match live AWS: `min_size=3`, `max_size=3`, warm pool `Running`.
  2. Reconciled sandbox config to current partial-decommissioned state: removed deleted app-tier ASGs from active `.tf`, updated launch template AMIs/default versions to live values, and kept live `365` day retention for `smoke-api-sandbox-pm2-out-log`.
  3. Removed stale dev/QA Secrets Manager import/resource config for deleted secrets.
  4. Moved decommissioned dev/QA generated resource files to `.tf.decommissioned` so Terraform will not recreate those retired stacks while preserving the old imported config for reference.
- **Verification:**
  - `terraform validate` passes for prod, sandbox, dev, and QA.
  - `terraform plan -lock=false -input=false -no-color -detailed-exitcode` shows `No changes. Your infrastructure matches the configuration.` for prod, sandbox, dev, and QA.
- **Notes:** UAT was not included in this cleanup pass and may still need a separate decommissioned-state reconciliation. Prod API ASG config was later updated again on 2026-04-29 to prepare the approved autoscaling change.

---

### 2026-04-29 — Prod API autoscaling applied
- **Approval reference:** User requested "proceed to autoscaling of prod" and then confirmed "applied"
- **Applied by:** Applied externally by user/operator; Codex prepared Terraform config and verified afterward
- **Execution location:** local repo — `code/infra/terraform/environments/prod/generated_prod.tf`
- **Systems affected:** `aws_autoscaling_group.prod_api_server` (`CodeDeploy_Smoke-API_d-NNRF41DGD`) and its warm pool
- **Summary:** Updated prod API ASG autoscaling bounds from fixed `min=max=3` to `min_size=2`, `max_size=5`, with warm pool `pool_state="Stopped"`. `desired_capacity` remains ignored in Terraform so the target tracking policy can own runtime capacity.
- **Verification:** `aws autoscaling describe-auto-scaling-groups` shows live `Desired=3`, `Min=2`, `Max=5` with three healthy InService instances. `aws autoscaling describe-warm-pool` shows warm pool config `PoolState=Stopped`, `MinSize=1`, `InstanceReusePolicy.ReuseOnScaleIn=true`. `terraform plan -lock=false -input=false -no-color -detailed-exitcode` for prod shows `No changes. Your infrastructure matches the configuration.`
- **Notes:** Codex did not run `terraform apply`; the remote change was applied externally and verified from the user's confirmation plus read-only AWS/Terraform checks.

---

### 2026-04-29 — OPT-001 sandbox Terraform decommission preparation applied
- **Approval reference:** User approved OPT-001 with "proceed", asked whether it could be done via Terraform, then confirmed "applied" after the stage-1 Terraform prepare diff was provided.
- **Applied by:** Applied externally by user/operator; Codex prepared Terraform config and verified afterward.
- **Execution location:** local repo — `code/infra/terraform/environments/sandbox/{data.tf,loadbalancer.tf,variables.tf}`
- **Systems affected:** `aws_db_instance.sensor_sandbox`, `aws_elasticache_replication_group.smokealarmsandbox`, `aws_lb.sandbox_api_lb`, `aws_lb.sandbox_sso_lb`
- **Summary:** Prepared sandbox fixed-cost resources for later Terraform decommission by disabling RDS and ALB deletion protection, setting RDS final snapshot behavior, preserving RDS automated backups, and setting final snapshot identifiers for RDS and Redis.
- **Verification:** `terraform plan -lock=false -input=false -no-color -detailed-exitcode` for sandbox shows `No changes. Your infrastructure matches the configuration.` `aws rds describe-db-instances` shows `sensor-sandbox` `DeletionProtection=false`. `aws elbv2 describe-load-balancer-attributes` shows both `Smoke-Sandbox-Api-LB` and `sandbox-sso-api` `deletion_protection.enabled=false`.
- **Notes:** Codex did not run `terraform apply`; the remote change was applied externally and verified from the user's confirmation plus read-only AWS/Terraform checks. This is stage 1 only; stage 2 will remove/gate the selected sandbox resources and produce the actual destroy plan.

---

### 2026-04-29 — OPT-001 sandbox EC2 decommission preparation applied
- **Approval reference:** User confirmed "applied" after the stage-2A Terraform prepare diff was provided.
- **Applied by:** Applied externally by user/operator; Codex prepared Terraform config and verified afterward.
- **Execution location:** local repo — `code/infra/terraform/environments/sandbox/compute.tf`
- **Systems affected:** `aws_instance.cron_sandbox`, `aws_instance.kafka_mongo_sandbox`, `aws_instance.emqx_sandbox`, `aws_instance.odoo_sandbox`
- **Summary:** Prepared stopped sandbox EC2 instances for later Terraform decommission by disabling API termination protection on Odoo/Kafka/EMQX and enabling root-volume delete-on-termination on cron, Kafka, and Odoo.
- **Verification:** `terraform plan -lock=false -input=false -no-color -detailed-exitcode` for sandbox shows `No changes. Your infrastructure matches the configuration.` `aws ec2 describe-instance-attribute` shows `DisableApiTermination=false` for `i-0b71daa2680a16a5d`, `i-07af1c06915741068`, and `i-0074106adaa006f2e`.
- **Notes:** Codex did not run `terraform apply`; the remote change was applied externally and verified from the user's confirmation plus read-only AWS/Terraform checks. This is stage 2A only; stage 2B will remove/gate the selected sandbox resources and produce the actual destroy plan.

---

### 2026-04-29 — OPT-001 sandbox fixed-cost decommission applied
- **Approval reference:** User requested "lets proceed with 2B", applied the Terraform stage externally, reported partial apply errors, then confirmed "this is done" after applying the finish plan.
- **Applied by:** Applied externally by user/operator; Codex prepared Terraform config/plans and verified afterward.
- **Execution location:** external Terraform apply from `code/infra/terraform/environments/sandbox`
- **Systems affected:** `sensor-sandbox` RDS, `smokealarmsandbox` ElastiCache, sandbox NAT Gateway `nat-0f00d13d55f11c828`, NAT EIP `eipalloc-0121eef619a926133`, sandbox API/SSO ALBs/listeners/target groups, stopped sandbox EC2 instances `i-0074106adaa006f2e`, `i-0b71daa2680a16a5d`, `i-008e2acd8997ff315`, `i-07af1c06915741068`, `i-0c84b9424bc1ce4dd`, and their Terraform-managed EIPs.
- **Summary:** Removed the selected sandbox fixed-cost resources from active Terraform and decommissioned them externally. The first apply partially completed and reported stale `ListenerNotFound` errors plus a NAT EIP association error; the follow-up finish plan removed the remaining unassociated NAT EIP.
- **Verification:** Final sandbox `terraform plan -lock=false -input=false -no-color -detailed-exitcode` shows `No changes. Your infrastructure matches the configuration.` Read-only AWS checks show `sensor-sandbox` and `smokealarmsandbox` are not found, NAT Gateway `nat-0f00d13d55f11c828` is `deleted`, NAT EIP `eipalloc-0121eef619a926133` is not found, the five stopped sandbox EC2 instances are terminated, the five sandbox instance EIPs are released, and the sandbox API/SSO ALBs are not found.
- **Notes:** Codex did not run `terraform apply` or destructive AWS APIs. Route53 sandbox records are outside this Terraform stage and may still need cleanup if they point at deleted ALBs or released EIPs. Stage 2B estimated gross recurring saving is about `USD 445/month` / `AUD 630/month` before taxes, discounts, partial-month timing, and residual snapshot or backup storage.

---

### 2026-05-06 — Production RDS storage gp3 optimisation applied externally
- **Approval reference:** User requested the RDS cost optimisation, ran the Terraform apply externally, and reported "applied".
- **Applied by:** Applied externally by user/operator; Codex prepared Terraform config/plans and verified read-only afterward.
- **Execution location:** external Terraform apply from `code/infra/terraform/environments/prod`
- **Systems affected:** `aws_db_instance.odoo_production` (`odoo-production`) and `aws_db_instance.sensor_prod` (`sensor-prod`)
- **Summary:** Production RDS storage-class optimisation moved `odoo-production` from `io1` to `gp3` and `sensor-prod` from `gp2` to `gp3` without changing instance class or Multi-AZ posture. Both Terraform resources omit explicit gp3 IOPS/throughput for sub-400 GiB RDS storage so AWS uses the included baseline.
- **Verification:** Read-only `aws rds describe-db-instances` after the external immediate apply showed both DB instances on current `StorageType=gp3`, `Iops=3000`, `StorageThroughput=125`, and no pending modified values. Both were in `storage-optimization`; RDS events showed each storage modification finished applying. `sensor-prod` CloudWatch `DatabaseConnections` stayed flat at 2 through the checked window.
- **Notes:** Codex did not run `terraform apply` or any mutating AWS API. Terraform prod defaults and optimisation varfiles were updated afterward so future plans do not try to revert `sensor-prod` to `gp2` or set invalid explicit gp3 throughput/IOPS for sub-400 GiB RDS instances.

---

### 2026-05-15 — Safer Ops production DNS alias applied externally
- **Approval reference:** User approved/proceeded with the Safer Ops production DNS Terraform step and later confirmed "deployed, looks like resolving and traffic is routing".
- **Applied by:** Applied externally by user/operator; Codex prepared the Terraform Route53 record and verified the record afterward.
- **Execution location:** external Terraform apply from `code/infra/terraform/environments/prod`
- **Systems affected:** Route53 public hosted zone `sensorglobal.com` (`Z03520551S3PY3J02L5ZN`) and Safer Ops production ALB `safer-ops-prod-854086076.ap-southeast-2.elb.amazonaws.com`
- **Summary:** Added `safer-ops.sensorglobal.com` as an `A` alias record to the Safer Ops production ALB.
- **Verification:** Read-only Route53 lookup showed `safer-ops.sensorglobal.com.` has an `A` alias target of `safer-ops-prod-854086076.ap-southeast-2.elb.amazonaws.com.` with hosted zone ID `Z1GM3OXH4ZPM65`.
- **Notes:** Codex did not run `terraform apply`, mutating AWS APIs, or live production HTTP/Kubernetes diagnostics. HTTPS traffic routing was confirmed by the user externally.
