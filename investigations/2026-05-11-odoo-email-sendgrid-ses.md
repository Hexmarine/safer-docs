# Odoo Email Sending Investigation

- Date: 2026-05-11
- Scope: Production Odoo email path after SensorGlobal moved backend transactional email from SendGrid to SES
- Related systems: Odoo 16, Postfix relay, SendGrid SMTP, Amazon SES, Route53

## Objective

Determine why Odoo email sends appear to be failing, and whether the failure is related to the recent SES cutover.

## Sources Used

- Repository files:
  - `code/odoo-configuration/compose/docker-compose.yml`
  - `code/odoo-configuration/Odoofile`
  - `code/odoo-configuration/README.md`
  - `code/odoo-addons/sg_website/controllers/main.py`
  - `docs/investigations/2026-04-21-odoo-integrations-analysis.md`
  - `docs/investigations/2026-05-09-sendgrid-template-migration.md`
  - `docs/runbooks/13-ses-sensorglobal-domain-provisioning.md`
- AWS read-only commands:
  - `aws ec2 describe-instances --region ap-southeast-2 --instance-ids i-0c5073e95aba77614`
  - `aws sesv2 get-account --region ap-southeast-2`
  - `aws sesv2 get-email-identity --region ap-southeast-2 --email-identity sensorglobal.com`
  - `aws route53 list-resource-record-sets --hosted-zone-id Z03520551S3PY3J02L5ZN`
  - approved SSM read-only diagnostics on `i-0c5073e95aba77614`

SSM diagnostics were run only after the user approved:

```text
Approved diagnostic window: Codex may run read-only SSM diagnostics on i-0c5073e95aba77614/Odoo-production for Odoo email delivery failure.
```

## Findings

Odoo email is independent from the Node backend email provider switch.

The production backend SES cutover uses `EMAIL_PROVIDER=ses` and SES SMTP settings in the backend runtime. Odoo does not use that path. The Odoo deployment uses its own Docker Compose/Postfix SMTP relay pattern.

The checked-in Odoo Compose file configures Postfix to relay through SendGrid SMTP:

```text
POSTFIX_RELAY_HOST=[smtp.sendgrid.net]:587
POSTFIX_RELAY_USER=apikey
POSTFIX_RELAY_PASSWORD=changeme
POSTFIX_FROM_NAME=Change Me
POSTFIX_FROM_ADDRESS=sensorglobal@change.me
```

The checked-in Odoo runtime config points Odoo SMTP at the local relay:

```text
smtp_server = "postfix-static-relay"
smtp_port = 25
```

The Odoo configuration README explains why this relay exists: many Odoo templates can emit dynamic user/company `From` addresses, and the relay rewrites the `From` header before forwarding to the external mail provider.

The addon code confirms this is still a relevant behavior. Many templates use expressions such as `object.user_id.email_formatted` or `user.email_formatted`, and one website flow explicitly looks up an Odoo mail server named like `sendgrid smtp` before sending a queued email.

SES itself is healthy in `ap-southeast-2`:

- `ProductionAccessEnabled=true`
- `SendingEnabled=true`
- `EnforcementStatus=HEALTHY`
- Daily quota: 50,000 messages
- Send rate: 14 messages/second
- `sensorglobal.com` identity is verified for sending
- SES DKIM status is `SUCCESS`
- Custom MAIL FROM `mail.sensorglobal.com` status is `SUCCESS`

Route53 contains both SES and legacy SendGrid email records:

- SES DKIM CNAMEs for `sensorglobal.com`
- SES custom MAIL FROM records for `mail.sensorglobal.com`
- Legacy SendGrid DKIM/CNAME records such as `s1`, `s2`, `prd`, `prd2`, `reply`, and compliance/reply subdomains

The production Odoo instance is running:

```text
Name: Odoo-production
InstanceId: i-0c5073e95aba77614
PrivateIp: 10.0.2.161
PublicIp: 54.253.158.197
State: running
```

Live host diagnostics confirmed the production configuration matches the repository pattern:

```text
Container: compose-postfix-static-relay-1
POSTFIX_RELAY_HOST=[smtp.sendgrid.net]:587
POSTFIX_FROM_NAME=Sensor Global
POSTFIX_FROM_ADDRESS=hello@sensorglobal.com

Container: data-odoo-16e
ODOOCONF__options__smtp_server=postfix-static-relay
ODOOCONF__options__email_from=donotreply@sensorglobal.com
ODOOCONF__options__db_name=sensorglobal_live
```

The live Odoo database initially had an active direct SMTP server record still pointing to SendGrid:

```text
(11, 'sendgrid smtp', 'smtp.sendgrid.net', 465, 'ssl', True)
(12, 'devnull', 'devnull.example.com', 25, 'none', True)
(10, 'Null', 'devnull.example.com', 25, 'none', False)
```

After the user/operator updated the Odoo UI on 2026-05-11, read-only verification showed the same compatibility-preserving record now points to SES:

```text
(11, 'sendgrid smtp', 'email-smtp.ap-southeast-2.amazonaws.com', 587, 'starttls', True)
(12, 'devnull', 'devnull.example.com', 25, 'none', True)
(10, 'Null', 'devnull.example.com', 25, 'none', False)
```

The user then sent an Odoo invitation email and confirmed it was received. This validates the primary Odoo UI outgoing-mail path after the `ir_mail_server` change.

The Odoo outbound mail table has a large failed/cancelled backlog:

```text
mail_mail state counts:
cancel     9734
exception 1806
sent       3300
```

The most recent outbound records are failing:

```text
54014 exception 2026-05-10 14:59:09 Connection unexpectedly closed
54012 exception 2026-05-09 14:57:09 Connection unexpectedly closed
54011 exception 2026-05-09 06:21:59 -5 No address associated with hostname
54010 exception 2026-05-08 14:58:46 Connection unexpectedly closed
54009 exception 2026-05-08 09:31:49 Connection unexpectedly closed
```

The Postfix relay queue was empty at the time of inspection, which suggests recent failures are being recorded in Odoo before or during SMTP handoff rather than accumulating as queued mail inside the relay.

## Current Assessment

The confirmed cause was that Odoo was still configured to send through old SendGrid SMTP paths while the rest of the platform had moved to SES.

This is consistent with the earlier SendGrid investigation: the current SendGrid API-key send path was not authorized for Mail Send API validation. The Odoo path is SMTP rather than API, so that exact API-key result does not prove Odoo SMTP is failing, but it increases the risk that SendGrid-backed transactional sending is no longer reliable.

SES is ready to accept mail for `sensorglobal.com`; the Odoo database mail server path has now been migrated to SES SMTP by the user/operator and verified by Codex read-only diagnostics.

There are two live Odoo mail paths to account for:

1. Host/container relay path: Odoo config points at `postfix-static-relay`, and the relay points to SendGrid.
2. Database mail server path: Odoo `ir_mail_server` had an active `sendgrid smtp` record pointing directly at SendGrid on port 465; this record now points to SES on port 587 with STARTTLS.

The database mail server path now uses SES. The host/container relay path remains a separate follow-up if any Odoo flows use the default `postfix-static-relay` instead of selecting the `sendgrid smtp` mail server record directly.

## Risks Or Constraints

- Do not change Odoo production host configuration without explicit approval.
- Do not restart Odoo or Postfix without an approved targeted production mutation.
- Do not print SMTP credentials in logs, docs, shell output, or chat.
- Directly switching Odoo to SES without preserving the relay's `From` rewrite may fail for templates that use dynamic user/company sender addresses.
- Some Odoo code still references a mail server named like `sendgrid smtp`, so the Odoo database configuration needs a controlled update or backwards-compatible naming decision after host-level relay migration.

## Recommended Next Steps

1. If needed, prepare an approved host-level Odoo relay migration for the remaining Postfix relay path:
   - keep Odoo pointing at `postfix-static-relay:25`
   - change Postfix relay host to `[email-smtp.ap-southeast-2.amazonaws.com]:587`
   - use SES SMTP username/password from approved secret storage
   - keep a static verified `From` address under `sensorglobal.com`

2. Monitor the next scheduled/customer-triggered Odoo emails for any fresh `mail_mail` exceptions.

3. After a successful host-level relay change, update the durable Odoo configuration repo and document the applied production change in `docs/applied-changes.md`.

## Open Questions

- Where should Odoo-specific SES SMTP credentials live: host environment, AWS Secrets Manager, or another existing Odoo deployment secret mechanism?
- Which sender address should the relay force for Odoo: `no-reply@sensorglobal.com`, `donotreply@sensorglobal.com`, or a CRM-specific address?
- Should the Odoo `ir_mail_server` keep the legacy name `sendgrid smtp` for compatibility while changing its host to SES, or should the custom code be updated first?
