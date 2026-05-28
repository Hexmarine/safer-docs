# SES Provisioning For `sensorglobal.com`

Date: 2026-05-10

Scope: prepare AWS SES in account `747293622182`, region `ap-southeast-2`, for the backend SES provider that was deployed dormant behind `EMAIL_PROVIDER`.

This runbook contains approval-gated AWS mutations. Codex did not execute these provisioning steps.

## Current State

Initial read-only discovery found:

- No SES configuration sets exist
- No dedicated IP pools exist
- No SES identities were found in `us-east-1`, `us-west-2`, `eu-west-1`, `eu-west-2`, or `ap-southeast-1`
- Route53 manages the `sensorglobal.com` public hosted zone: `Z03520551S3PY3J02L5ZN`
- `mail.sensorglobal.com` has no existing records and is available for custom MAIL FROM

Existing DNS relevant to email:

- Root SPF:
  - `v=spf1 include:spf.protection.outlook.com include:relay.mailchannels.net -all`
- DMARC:
  - `v=DMARC1; p=none; fo=1`
- Existing DKIM records are for SendGrid, Mailchimp, and Microsoft.

Current verified status after the 2026-05-10 operator-applied changes:

- SES production access is granted in `ap-southeast-2`: `ProductionAccessEnabled=true`
- SES sending is enabled and healthy: `SendingEnabled=true`, `EnforcementStatus=HEALTHY`
- Production quota is `50000` messages/day and `14` messages/second
- SES review status is `GRANTED`, case `177836478300309`
- The only SES identity in `ap-southeast-2` is now `sensorglobal.com`
- `sensorglobal.com` is verified for sending
- SES Easy DKIM is `SUCCESS`
- Custom MAIL FROM `mail.sensorglobal.com` is `SUCCESS`
- The production backend secret `sensor-prod` does not currently contain `EMAIL_PROVIDER`, `SES_HOST`, `SES_PORT`, `SES_USER`, or `SES_PASSWORD`

## Target Design

- SES identity: `sensorglobal.com`
- Region: `ap-southeast-2`
- DKIM: SES Easy DKIM CNAMEs under `sensorglobal.com`
- Custom MAIL FROM: `mail.sensorglobal.com`
- Backend remains dormant on SendGrid until cutover:
  - `EMAIL_PROVIDER` unset or `sendgrid`
- Cutover config later:
  - `EMAIL_PROVIDER=ses`
  - `SES_HOST=email-smtp.ap-southeast-2.amazonaws.com`
  - `SES_PORT=587`
  - `SES_USER=<smtp user>`
  - `SES_PASSWORD=<smtp password>`

Terraform now contains the durable SES identity and DNS resources in:

```text
code/infra/terraform/environments/account/ses.tf
```

Use Terraform for the SES identity, DKIM DNS records, and custom MAIL FROM records. Keep the SES production-access request and SMTP credential creation as separate operator steps.

## Operator Steps

### 1. Baseline

```bash
aws sesv2 get-account --region ap-southeast-2
aws sesv2 list-email-identities --region ap-southeast-2
aws route53 list-resource-record-sets \
  --hosted-zone-id Z03520551S3PY3J02L5ZN \
  --query "ResourceRecordSets[?Name == 'sensorglobal.com.' || Name == '_dmarc.sensorglobal.com.' || Name == 'mail.sensorglobal.com.']"
```

### 2. Create The SES Domain Identity And DNS Records

Preferred path: run the account Terraform environment after review/approval.

```bash
cd code/infra/terraform/environments/account
terraform plan
```

Expected resources:

- `aws_sesv2_email_identity.sensorglobal_com`
- `aws_route53_record.sensorglobal_com_ses_dkim[0..2]`
- `aws_sesv2_email_identity_mail_from_attributes.sensorglobal_com`
- `aws_route53_record.sensorglobal_com_ses_mail_from_mx`
- `aws_route53_record.sensorglobal_com_ses_mail_from_spf`

Apply only after approval:

```bash
terraform apply
```

Manual CLI fallback:

```bash
aws sesv2 create-email-identity \
  --email-identity sensorglobal.com \
  --region ap-southeast-2
```

Then fetch the DKIM tokens:

```bash
aws sesv2 get-email-identity \
  --email-identity sensorglobal.com \
  --region ap-southeast-2
```

Record the three values under `DkimAttributes.Tokens`.

### 3. Add DKIM DNS Records

For each SES DKIM token, add this Route53 CNAME:

```text
<token>._domainkey.sensorglobal.com.  CNAME  <token>.dkim.amazonses.com.
```

Use this Route53 change shape, replacing the three token placeholders:

```json
{
  "Comment": "Verify SES Easy DKIM for sensorglobal.com",
  "Changes": [
    {
      "Action": "UPSERT",
      "ResourceRecordSet": {
        "Name": "<token-1>._domainkey.sensorglobal.com.",
        "Type": "CNAME",
        "TTL": 600,
        "ResourceRecords": [{ "Value": "<token-1>.dkim.amazonses.com." }]
      }
    },
    {
      "Action": "UPSERT",
      "ResourceRecordSet": {
        "Name": "<token-2>._domainkey.sensorglobal.com.",
        "Type": "CNAME",
        "TTL": 600,
        "ResourceRecords": [{ "Value": "<token-2>.dkim.amazonses.com." }]
      }
    },
    {
      "Action": "UPSERT",
      "ResourceRecordSet": {
        "Name": "<token-3>._domainkey.sensorglobal.com.",
        "Type": "CNAME",
        "TTL": 600,
        "ResourceRecords": [{ "Value": "<token-3>.dkim.amazonses.com." }]
      }
    }
  ]
}
```

Apply it:

```bash
aws route53 change-resource-record-sets \
  --hosted-zone-id Z03520551S3PY3J02L5ZN \
  --change-batch file://ses-dkim-sensorglobal-com.json
```

### 4. Configure Custom MAIL FROM

Use `mail.sensorglobal.com` so SPF can align with the root `sensorglobal.com` domain without changing the existing root SPF record used by Microsoft and MailChannels.

```bash
aws sesv2 put-email-identity-mail-from-attributes \
  --email-identity sensorglobal.com \
  --mail-from-domain mail.sensorglobal.com \
  --behavior-on-mx-failure USE_DEFAULT_VALUE \
  --region ap-southeast-2
```

Add these DNS records:

```text
mail.sensorglobal.com.  MX   10 feedback-smtp.ap-southeast-2.amazonses.com.
mail.sensorglobal.com.  TXT  "v=spf1 include:amazonses.com -all"
```

Route53 change shape:

```json
{
  "Comment": "Configure SES custom MAIL FROM for sensorglobal.com",
  "Changes": [
    {
      "Action": "UPSERT",
      "ResourceRecordSet": {
        "Name": "mail.sensorglobal.com.",
        "Type": "MX",
        "TTL": 600,
        "ResourceRecords": [{ "Value": "10 feedback-smtp.ap-southeast-2.amazonses.com." }]
      }
    },
    {
      "Action": "UPSERT",
      "ResourceRecordSet": {
        "Name": "mail.sensorglobal.com.",
        "Type": "TXT",
        "TTL": 600,
        "ResourceRecords": [{ "Value": "\"v=spf1 include:amazonses.com -all\"" }]
      }
    }
  ]
}
```

Apply it:

```bash
aws route53 change-resource-record-sets \
  --hosted-zone-id Z03520551S3PY3J02L5ZN \
  --change-batch file://ses-mail-from-sensorglobal-com.json
```

### 5. Wait For Verification

Poll until `VerifiedForSendingStatus=true` and DKIM `Status=SUCCESS`:

```bash
aws sesv2 get-email-identity \
  --email-identity sensorglobal.com \
  --region ap-southeast-2
```

DNS propagation may take minutes. If verification does not complete, check for typoed token names, missing trailing dots, or conflicting DNS records.

### 6. Request SES Production Access

Use the AWS SES console or `put-account-details`. Example content for the request:

- Mail type: transactional
- Website URL: `https://sensorglobal.com`
- Use case: Sensor Global backend sends account, job, property, invoice, and compliance workflow notifications to customers and operational users.
- Bounce/complaint handling: SES account-level suppression is enabled for `BOUNCE` and `COMPLAINT`; event destination can be added before higher-volume rollout.
- Recipient acquisition: recipients are Sensor Global customers, agencies, contractors, tenants, property owners, and staff created through the platform workflows.
- Unsubscribe: not used for transactional messages; marketing mail should remain outside this migration unless explicitly reviewed.

```bash
aws sesv2 put-account-details \
  --production-access-enabled \
  --mail-type TRANSACTIONAL \
  --website-url https://sensorglobal.com \
  --additional-contact-email-addresses '<ops-email@example.com>' \
  --contact-language EN \
  --use-case-description 'Sensor Global backend transactional notifications for account, job, property, invoice, and compliance workflows.' \
  --region ap-southeast-2
```

Poll:

```bash
aws sesv2 get-account --region ap-southeast-2
```

Pass condition:

- `ProductionAccessEnabled=true`
- Send quota raised above sandbox level

### 7. Create SMTP Credentials

Create credentials only after identity verification is complete. Do not commit or print the password in docs or tickets.

Preferred storage target is the same production secret/config mechanism that currently stores backend environment variables.

Recommended IAM principal:

```text
sensor-backend-ses-smtp
```

The credential needs permission only for SES email sending. Keep it scoped to the verified identity in `ap-southeast-2`:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["ses:SendEmail", "ses:SendRawEmail"],
      "Resource": "arn:aws:ses:ap-southeast-2:747293622182:identity/sensorglobal.com"
    }
  ]
}
```

Preferred operator path:

1. Create SES SMTP credentials in the SES console for the verified `ap-southeast-2` region.
2. Name the IAM user `sensor-backend-ses-smtp`.
3. Store the generated SMTP username and SMTP password directly in Secrets Manager.
4. Do not paste the generated password into chat, tickets, docs, shell history, or repo files.

CLI fallback if using IAM access keys:

1. Create the IAM user and attach the restricted inline policy above.
2. Create one access key.
3. Convert the IAM secret access key to an SES SMTP password for `ap-southeast-2` using the AWS documented SMTP password algorithm.
4. Store the access key id as `SES_USER` and the converted SMTP password as `SES_PASSWORD`.

Required backend config:

```env
SES_HOST=email-smtp.ap-southeast-2.amazonaws.com
SES_PORT=587
SES_USER=<smtp user>
SES_PASSWORD=<smtp password>
EMAIL_PROVIDER=sendgrid
```

Keep `EMAIL_PROVIDER=sendgrid` until the cutover window. Adding the dormant `SES_*` values alone should not switch sending to SES.

Safe operator merge pattern for `sensor-prod` that avoids putting the SMTP password in shell history or command arguments:

```bash
set -euo pipefail

current="$(mktemp)"
next="$(mktemp)"
ses_user_file="$(mktemp)"
ses_password_file="$(mktemp)"
trap 'rm -f "$current" "$next" "$ses_user_file" "$ses_password_file"' EXIT

read -r -p "SES SMTP username: " ses_user
printf '%s' "$ses_user" > "$ses_user_file"
unset ses_user

read -r -s -p "SES SMTP password: " ses_password
printf '\n'
printf '%s' "$ses_password" > "$ses_password_file"
unset ses_password

aws secretsmanager get-secret-value \
  --secret-id sensor-prod \
  --region ap-southeast-2 \
  --output json \
  | jq '.SecretString | if type == "string" then fromjson else . end' > "$current"

jq \
  --rawfile ses_user "$ses_user_file" \
  --rawfile ses_password "$ses_password_file" \
  '. + {
    SES_HOST: "email-smtp.ap-southeast-2.amazonaws.com",
    SES_PORT: "587",
    SES_USER: $ses_user,
    SES_PASSWORD: $ses_password,
    EMAIL_PROVIDER: "sendgrid"
  }' "$current" > "$next"

aws secretsmanager put-secret-value \
  --secret-id sensor-prod \
  --region ap-southeast-2 \
  --secret-string "file://$next"
```

Verification without printing values:

```bash
aws secretsmanager get-secret-value \
  --secret-id sensor-prod \
  --region ap-southeast-2 \
  --output json \
  | jq -r '.SecretString | if type == "string" then fromjson else . end
    | {
        EMAIL_PROVIDER,
        SES_HOST,
        SES_PORT,
        has_SES_USER: has("SES_USER"),
        has_SES_PASSWORD: has("SES_PASSWORD")
      }'
```

Expected before cutover:

```json
{
  "EMAIL_PROVIDER": "sendgrid",
  "SES_HOST": "email-smtp.ap-southeast-2.amazonaws.com",
  "SES_PORT": "587",
  "has_SES_USER": true,
  "has_SES_PASSWORD": true
}
```

### 8. Optional: Create SES Configuration Set

Before high-volume rollout, create a configuration set and event destination for bounces, complaints, deliveries, and rejects. This is not required for the first controlled test, but it improves observability.

Suggested name:

```text
sensor-backend-transactional
```

### 9. Pre-Cutover Verification

Run read-only checks:

```bash
aws sesv2 get-account --region ap-southeast-2
aws sesv2 get-email-identity --email-identity sensorglobal.com --region ap-southeast-2
aws ses get-send-quota --region ap-southeast-2
```

Expected:

- `ProductionAccessEnabled=true`
- `VerifiedForSendingStatus=true`
- DKIM `Status=SUCCESS`
- Custom MAIL FROM attributes show `mail.sensorglobal.com`
- Quota is production-level

Backend should already be deployed with SES support but still using SendGrid.

### 10. Cutover

During a quiet window, update backend runtime config:

```env
EMAIL_PROVIDER=ses
```

Then deploy/restart using the normal backend process. The backend reads config at process start, so changing Secrets Manager alone is not enough for running PM2 processes.

Smoke one or two safe email flows with a controlled recipient. Watch:

- backend logs for SES SMTP errors
- SES `Send`, `Reject`, `Bounce`, and `Complaint` metrics
- received email headers for DKIM/SPF/DMARC pass
- app message-id/audit-log behavior

### 11. Rollback

Set:

```env
EMAIL_PROVIDER=sendgrid
```

Then redeploy/restart using the normal backend process.

Rollback does not require removing SES DNS records or identities.

## Notes

- Do not delete SendGrid DNS records during SES rollout. They are still needed until SendGrid is retired deliberately.
- Do not change root `sensorglobal.com` SPF unless a specific deliverability review says to. Custom MAIL FROM keeps SES SPF isolated on `mail.sensorglobal.com`.
- The previous failed SES identity `api.sensorglobal.com` was removed after `sensorglobal.com` was verified. This did not change the API DNS record.
- Do not record SMTP passwords in this repo.
