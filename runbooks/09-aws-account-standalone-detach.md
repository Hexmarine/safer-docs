# Runbook 09 — AWS Account Standalone Detach

**Applies to:** AWS account `747293622182`  
**Goal:** Leave the Ingram Micro parent AWS Organization and continue as a standalone JV-paid AWS account  
**Risk:** Medium to high — workloads stay in place, but billing, support, org guardrails, and legal/security controls change  
**Approval required:** Yes — explicit written approval immediately before the detach command  
**Expected detach path:** member account runs `aws organizations leave-organization`  
**Fallback detach path:** Ingram Micro management account removes this member account

---

## Operating Rules

- Treat every AWS command as production-impacting unless proven read-only.
- All checks before Step 5 are discovery or console/account-readiness checks.
- Do not run `aws organizations leave-organization` until every required gate below is marked PASS and a final written approval is recorded.
- Do not run any parent-side removal command from this repo. If fallback is needed, Ingram Micro should run it from their management account.
- Record only the final executed detach and confirmed results in `docs/applied-changes.md`; do not log preparation tasks there.

---

## Step 0 — Scope Confirmation

Required before starting verification.

- [ ] **G0.1 — End state confirmed**
  - Expected answer: account `747293622182` becomes standalone and is paid directly by the JV.
  - Not in scope: workload migration, immediate new AWS Organization, new account buildout.
  - Evidence:
  - Owner:

- [ ] **G0.2 — Execution path confirmed**
  - Expected path: member-account initiated `leave-organization`.
  - Fallback: Ingram Micro removes the account only if `leave-organization` is blocked.
  - Evidence: 2026-04-28 IAM simulation for `arn:aws:iam::747293622182:user/peresada1@gmail.com` returned `allowed` for `organizations:LeaveOrganization` when `aws:MultiFactorAuthPresent=true`; without MFA context it returned `explicitDeny` via the account's self-managed MFA policies. This confirms IAM permission with MFA, but does not prove the real call will pass SCP or standalone-account readiness checks.
  - Owner: TBD

- [ ] **G0.3 — Change window agreed**
  - Recommended window: low-traffic business window with billing/admin owners available.
  - Evidence:
  - Owner:

---

## Step 1 — Member Account Readiness

These gates make sure AWS will allow the account to operate standalone.

- [ ] **G1.1 — Root user ownership recovered and confirmed**
  - Current issue: the known person with root access has left the company; the active IAM admin account is the only confirmed access path.
  - This is a pre-detach blocker unless leadership explicitly accepts operating without root ownership, which is not recommended for a standalone production account.
  - Current CLI evidence:
    - `aws account get-primary-email --account-id 747293622182` from `arn:aws:iam::747293622182:user/peresada1@gmail.com` returned `AccessDeniedException` on 2026-04-28.
    - This member admin cannot retrieve or update the root/primary email through the Account API.
  - Preferred recovery path while the account is still under Ingram Micro:
    1. Ask Ingram Micro / the Organizations management account to retrieve the current root/primary email for account `747293622182`.
    2. Ask them to update the root/primary email to a JV-controlled group mailbox using AWS Organizations / Account Management.
    3. Complete the OTP verification sent to the new mailbox.
    4. Use the new root email to reset the root password if the password is unknown.
    5. Recover or reset root MFA if the old device is unavailable, using AWS account recovery / Support flow as needed.
    6. After recovery, enable root MFA on a JV-controlled device and store recovery ownership in the JV access process.
  - Verification once recovered:
    1. Sign in as root for account `747293622182`.
    2. Confirm root email inbox is JV-controlled.
    3. Confirm root MFA is enabled and controlled by the JV.
    4. Confirm account recovery phone number and primary contact are current.
    5. Sign out; do not use root for normal operations.
  - Recovery options if the current root holder is unavailable:
    - Best: Ingram Micro updates the member account root/primary email centrally before detach.
    - Good: company IT recovers the existing root mailbox if it is a company-controlled address.
    - Slower: AWS Support account recovery using company/legal/billing proof.
    - Risk acceptance only: detach with IAM admin access and recover root later; not recommended for a standalone production account because parent-side recovery assistance is lost after detach.
  - If leadership accepts "recover root later" after detach:
    1. Record explicit risk acceptance before Step 5, including that root email and root MFA are not currently controlled by the JV.
    2. Confirm at least two JV-controlled IAM administrator access paths with MFA exist before detach. Do not rely on a single human login.
    3. Keep Ingram Micro available during the detach window in case `leave-organization` is blocked and parent-side removal is required.
    4. After detach, search for the root/primary email through company email archives, password vaults, invoices, old AWS notifications, and former-admin handover records.
    5. If the root email is known and the mailbox is controlled, use root "Forgot password" and complete root MFA recovery if needed.
    6. If the root email is known but the mailbox is not controlled, recover the mailbox through company IT if it is on a company domain, or start AWS sign-in/account recovery if it is not.
    7. If the root email is unknown, use AWS sign-in/account recovery and company/legal/billing evidence; expect this to be slower and less deterministic than the pre-detach Ingram route.
    8. Once recovered, update the root email to a JV-controlled group mailbox, enable root MFA, remove any root access keys if present, and document the root recovery process.
  - Evidence to record: current root email owner or replacement mailbox, date changed/recovered, whether root MFA is enabled, who owns recovery process. Do not record passwords, MFA seed, backup codes, OTPs, or screenshots containing secrets.
  - Evidence: Not recovered before detach. User confirmed approval to proceed with recover-later risk; additional admin account was created before detach. Post-detach `account:GetPrimaryEmail` from `arn:aws:iam::747293622182:user/peresada1@gmail.com` still returns `AccessDeniedException`.
  - Owner: JV / platform owner

- [x] **G1.2 — Primary account contact confirmed**
  - Read-only CLI:
    ```bash
    aws account get-contact-information --output json
    ```
  - Expected: primary contact is current and belongs to the JV/account owner.
  - Evidence: 2026-04-28 post-detach `account get-contact-information` returns `Sensor Global Pty Ltd`, `7/3 Koala Crescent`, West Gosford NSW 2250, AU, phone `+61418652822`.
  - Owner: JV / billing owner

- [ ] **G1.3 — Alternate contacts configured**
  - Required contacts: billing, operations, security.
  - Read-only CLI:
    ```bash
    aws account get-alternate-contact --alternate-contact-type BILLING --output json
    aws account get-alternate-contact --alternate-contact-type OPERATIONS --output json
    aws account get-alternate-contact --alternate-contact-type SECURITY --output json
    ```
  - Current known state from 2026-04-28: all three were missing.
  - Evidence: 2026-04-28 post-detach: operations and security contacts are configured for Yevgen Peresada; billing alternate contact still missing (`ResourceNotFoundException`).
  - Owner: JV / billing owner

- [x] **G1.4 — Payment method configured**
  - Confirm in AWS Billing console that the JV payment method is active.
  - CLI coverage is limited; use console evidence.
  - Evidence: User confirmed in chat on 2026-04-28: "added new payment method". CLI cannot display full payment-method state; console confirmation is treated as evidence.
  - Owner: User / JV billing owner

- [ ] **G1.5 — Phone verification / standalone sign-up completed**
  - Confirm AWS does not prompt for missing standalone-account information.
  - Evidence:
  - Owner:

- [ ] **G1.6 — Support plan selected**
  - Confirm support plan suitable for production operations.
  - Read-only signal:
    ```bash
    aws support describe-severity-levels --region us-east-1 --output json
    ```
  - Current known state from 2026-04-28: Support API returned `SubscriptionRequiredException`.
  - Evidence: 2026-04-28 post-detach: `aws support describe-severity-levels --region us-east-1` still returns `SubscriptionRequiredException`; Premium/Business Support API access is not active.
  - Owner: JV / billing owner

---

## Step 2 — Parent / Ingram Evidence Pack

These gates close the information we cannot see from the member account.

- [ ] **G2.1 — OU path exported**
  - Ask Ingram Micro for the OU path containing account `747293622182`.
  - Evidence:
  - Owner:

- [ ] **G2.2 — SCPs exported**
  - Ask for direct and inherited SCP names plus JSON policy documents.
  - Reason: after detach, these controls disappear.
  - Evidence:
  - Owner:

- [ ] **G2.3 — Trusted-access services exported**
  - Ask for enabled AWS Organizations trusted-access services.
  - Examples to check: Security Hub, GuardDuty, Config, IAM Access Analyzer, Backup, IAM Identity Center.
  - Evidence:
  - Owner:

- [ ] **G2.4 — Delegated administrator relationships exported**
  - Confirm whether `747293622182` is a delegated admin or managed member for any org-integrated service.
  - Evidence:
  - Owner:

- [ ] **G2.5 — Organization account tags exported**
  - AWS Organizations account tags can be lost on removal; export if they matter.
  - Evidence:
  - Owner:

- [ ] **G2.6 — Billing/reseller/support commitments confirmed**
  - Confirm any consolidated billing, reseller, Support, Marketplace, Savings Plan, RI, or private offer commitments tied to Ingram Micro.
  - Evidence:
  - Owner:

- [ ] **G2.7 — AWS Artifact / legal agreement coverage checked**
  - Confirm whether organization-level agreements currently cover this account and whether the JV must accept replacements.
  - Evidence:
  - Owner:

---

## Step 3 — Account Snapshot And Exports

These gates preserve evidence before the organization boundary changes.

- [x] **G3.1 — Identity and organization baseline captured**
  - Read-only CLI:
    ```bash
    aws sts get-caller-identity --output json
    aws organizations describe-organization --output json
    aws iam get-account-summary --output json
    ```
  - Evidence: 2026-04-28 post-detach: `sts get-caller-identity` confirms account `747293622182`, principal `arn:aws:iam::747293622182:user/peresada1@gmail.com`; `organizations describe-organization` returns `AWSOrganizationsNotInUseException`; `iam get-account-summary` returns 27 users, 191 roles, 139 policies, 20 MFA devices, 13 MFA devices in use.
  - Owner: Codex verified read-only

- [ ] **G3.2 — Cost and billing history exported**
  - Export or screenshot:
    - Cost Explorer monthly service costs
    - Bills/invoices
    - CUR setup if available
    - Marketplace/subscription charges
  - Reason: Cost Explorer visibility can change after org membership changes.
  - Minimum acceptable export:
    1. In Cost Explorer, export monthly unblended cost by service for the last 6-12 months.
    2. In Bills, save the latest invoices / bill details available to this member account.
    3. In Cost Explorer, export Marketplace or subscription-style charges if visible.
    4. Save a short note saying whether CUR exists and, if so, where it is delivered.
  - CLI helper for monthly service cost, if Cost Explorer access is available:
    ```bash
    aws ce get-cost-and-usage \
      --time-period Start=2026-01-01,End=2026-05-01 \
      --granularity MONTHLY \
      --metrics UnblendedCost \
      --group-by Type=DIMENSION,Key=SERVICE \
      --output json > tmp/cost-by-service-before-detach-$(date +%Y%m%d).json
    ```
  - Treat console exports/screenshots as acceptable if CLI access is denied or incomplete.
  - Evidence:
  - Owner:

- [ ] **G3.3 — Route 53 domain renewal safety confirmed**
  - Confirm AutoRenew and payment safety for all Route 53 Registrar domains.
  - Suggested read-only CLI:
    ```bash
    aws route53domains list-domains --region us-east-1 --output table
    ```
  - Evidence:
  - Owner:

- [ ] **G3.4 — Service quota snapshot captured**
  - Focus services: EC2, ELB, VPC/EIP/NAT, RDS, ElastiCache, CloudFront, Route 53, ACM, WAF, S3, messaging/SMS.
  - Console export is acceptable.
  - Evidence:
  - Owner:

- [ ] **G3.5 — IAM high-risk snapshot captured**
  - Read-only starting points:
    ```bash
    aws iam get-account-summary --output json
    aws iam list-users --output table
    aws iam list-roles --output table
    aws iam list-account-aliases --output json
    ```
  - Minimum review: admin users, active access keys, parent/reseller roles, old vendor users.
  - Evidence:
  - Owner:

---

## Step 4 — Replacement Guardrails

These gates reduce the risk from SCPs and org services disappearing.

- [ ] **G4.1 — SCP replacement decision recorded**
  - For each exported SCP, record one of:
    - recreate as local IAM/permission-boundary control
    - accept removal
    - defer with owner and deadline
  - Evidence:
  - Owner:

- [ ] **G4.2 — Security service continuity confirmed**
  - Confirm standalone behavior or replacement ownership for:
    - CloudTrail
    - Security Hub
    - GuardDuty
    - AWS Config
    - IAM Access Analyzer
    - AWS Backup
    - CloudWatch alarms/logs
  - Evidence:
  - Owner:

- [ ] **G4.3 — Residual parent/reseller access plan agreed**
  - Confirm which, if any, Ingram Micro access remains after detach.
  - Current known state: `OrganizationAccountAccessRole` was not found, but IAM should still be reviewed.
  - Evidence:
  - Owner:

- [x] **G4.4 — Production health baseline captured**
  - Use existing health check if available:
    ```bash
    bash scripts/health-check.sh --save-baseline tmp/baseline-before-account-detach-$(date +%Y%m%d-%H%M%S).json
    ```
  - Do not proceed if production has unexplained failures.
  - What this captures: ALB health and 5xx rate, RDS CPU/memory/storage/connections, EC2 status, CloudWatch alarms, SMS sends, and other platform checks implemented by `scripts/health-check.sh`.
  - If `HEALTH_API_KEY` is not available, the HTTP application health check may warn/skip, but AWS infrastructure metrics should still be useful. Record any skipped checks explicitly.
  - Minimum acceptable baseline if the script cannot run:
    ```bash
    aws elbv2 describe-target-health --target-group-arn <prod-api-target-group-arn> --output table
    aws rds describe-db-instances --db-instance-identifier odoo-production --output table
    aws cloudwatch describe-alarms --state-value ALARM --region ap-southeast-2 --output table
    aws ec2 describe-instance-status --include-all-instances --region ap-southeast-2 --output table
    ```
  - Evidence: 2026-04-28 post-detach health baseline saved to `tmp/baseline-after-account-detach-20260428-144824.json`; ALB healthy hosts 3, ALB 5xx 0.0%, RDS CPU 5.5%, RDS free memory 1933 MB, RDS free storage 90.0 GB, RDS connections 15, EC2 status 7/7 ok, SMS sends last 24h 107. Backend API HTTP check skipped because `HEALTH_API_KEY` was not set. 7 CloudWatch alarms were already firing/informational in this report.
  - Owner: Codex verified read-only

---

## Step 5 — Final Approval And Detach

Hard stop until every prior required gate is PASS.

- [ ] **G5.1 — Final go/no-go completed**
  - Required attendees/owners:
    - JV account owner
    - billing owner
    - production/platform owner
    - security/contact owner
    - Ingram Micro contact on standby if fallback is needed
  - Evidence:
  - Owner:

- [x] **G5.2 — Final written approval recorded**
  - Approval must explicitly authorize member-account detach for account `747293622182`.
  - Evidence: User confirmed in chat on 2026-04-28: "ok, got approval. created additional account. did detach".
  - Owner: User / JV

- [x] **G5.3 — Execute member-account detach**
  - Mutating command:
    ```bash
    aws organizations leave-organization
    ```
  - Expected outcome: account becomes standalone.
  - Evidence: User confirmed detach was executed. Post-detach `aws organizations describe-organization` returns `AWSOrganizationsNotInUseException: Your account is not a member of an organization.`
  - Owner: User / JV

- [ ] **G5.4 — Fallback if member detach is blocked**
  - If `leave-organization` is denied by SCP or permissions, ask Ingram Micro to run from management account:
    ```bash
    aws organizations remove-account-from-organization --account-id 747293622182
    ```
  - Evidence:
  - Owner:

---

## Step 6 — Immediate Post-Detach Validation

Run these checks immediately after detach.

- [x] **G6.1 — Organization membership state confirmed**
  - Read-only CLI:
    ```bash
    aws organizations describe-organization --output json
    ```
  - Expected for standalone: command should no longer show membership for this account; exact error/output depends on AWS state and permissions.
  - Evidence: 2026-04-28 `aws organizations describe-organization --output json` returned `AWSOrganizationsNotInUseException: Your account is not a member of an organization.`
  - Owner: Codex verified read-only

- [ ] **G6.2 — Billing and support confirmed**
  - Confirm billing console shows JV-owned payment setup.
  - Confirm support plan remains active.
  - Evidence: Payment method confirmed by user in console on 2026-04-28. Primary account contact reads `Sensor Global Pty Ltd`, West Gosford NSW. Billing alternate contact still missing (`ResourceNotFoundException`). Support API still returns `SubscriptionRequiredException`; support plan requires console confirmation if Business/Premium support is desired.
  - Owner: JV / billing owner

- [x] **G6.3 — Production health baseline rechecked**
  - Compare against pre-detach baseline:
    ```bash
    bash scripts/health-check.sh --compare-to tmp/baseline-before-account-detach-*.json
    ```
  - Evidence: 2026-04-28 health check passed core infra metrics: ALB healthy hosts 3, ALB 5xx 0.0%, RDS CPU 5.5%, RDS free memory 1933 MB, RDS free storage 90.0 GB, RDS connections 15, EC2 7/7 ok, SMS sends last 24h 107. Backend API HTTP check skipped due to missing `HEALTH_API_KEY`; 7 existing CloudWatch alarms reported informationally.
  - Owner: Codex verified read-only

- [x] **G6.4 — Route 53 and domain renewal safety rechecked**
  - Confirm domains remain visible and AutoRenew/payment path is safe.
  - Evidence: 2026-04-28 `route53domains list-domains` shows all 14 domains visible and `AutoRenew=True`. Near renewals remain: `sensorglobal.au` and `sensorglobal.net` in June 2026; `sensorglobal.co.uk` and `sensorglobal.com.au` in July 2026; `sensorinsure.com` and `.com.au` in September 2026.
  - Owner: Codex verified read-only

- [ ] **G6.5 — Security services rechecked**
  - Confirm CloudTrail, CloudWatch alarms/logging, Security Hub/GuardDuty/Config if used.
  - Evidence: Partially checked via health script: CloudWatch alarms/logging signals remain queryable. Deeper CloudTrail, Security Hub, GuardDuty, Config, Access Analyzer, and Backup standalone behavior still pending.
  - Owner: TBD

- [ ] **G6.6 — IAM permission broadening reviewed**
  - Re-check high-privilege users/roles after SCP removal.
  - Remove or constrain any access that was previously relying on SCP denial.
  - Evidence:
  - Owner:

- [ ] **G6.7 — Residual Ingram access removed or explicitly retained**
  - Confirm any parent/reseller IAM role/user/trust is either removed or documented as intentionally retained.
  - Evidence:
  - Owner:

- [x] **G6.8 — Applied change logged**
  - After successful detach, add an entry to `docs/applied-changes.md` with:
    - date
    - approval reference
    - who executed the detach
    - systems affected
    - verification notes
  - Evidence: `docs/applied-changes.md` updated with 2026-04-28 standalone detach entry.
  - Owner: Codex

---

## Stop Conditions

Stop before detach if any of these are true:

- Root access or payment method is not confirmed.
- AWS still prompts for missing standalone-account setup.
- Support plan decision is unresolved.
- Ingram Micro cannot provide SCP/org context and the team has not accepted that risk in writing.
- Production health baseline has unexplained failures.
- Route 53 domain renewal/payment safety is unresolved.
- Final written approval is missing.
- `leave-organization` is blocked and Ingram Micro is not available for fallback removal.
