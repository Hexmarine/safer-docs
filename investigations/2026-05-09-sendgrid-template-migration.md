# SendGrid Template Migration Investigation

**Date:** 2026-05-09  
**Environment:** Production backend API (`sensor-alarm-backend`)  
**Related system:** SendGrid transactional email  
**Status:** Read-only investigation complete; migration not yet implemented

---

## Summary

The production SendGrid account is still reachable with the API keys stored in AWS Secrets Manager secret `sensor-prod`, but the keys have different capabilities:

- `SENDGRID_API_KEY_FOR_READING` can read dynamic template content.
- `SENDGRID_API_KEY` can read account/webhook metadata but is **not authorized to send mail** through the SendGrid Mail Send API.
- Both explicit SendGrid API keys failed sandbox-mode mail-send validation with `Authenticated user is not authorized to send mail`.

The production backend currently sends email through `MailManager.sendMail()`, which uses `CONFIG.APP.SENDGRID_API_KEY`. Unless production has another runtime override not visible in `sensor-prod`, backend SendGrid email sends are likely failing or silently no-oping.

The old SMTP-style `EMAIL_PASSWORD` value exists, but this investigation did not transmit it to SendGrid as a bearer token because it is not explicitly a SendGrid API key. The active `sendMail()` path does not call the legacy Nodemailer/SMTP methods.

---

## Template Access Check

Read-only SendGrid API probes confirmed:

| Check | Result |
|---|---|
| `sensor-prod` contains `SENDGRID_API_KEY` | Present |
| `sensor-prod` contains `SENDGRID_API_KEY_FOR_READING` | Present |
| List dynamic templates with reading key | HTTP 200 |
| Fetch known production template `S003` | HTTP 200; active HTML version present |
| Production template mappings checked | 148 |
| Mapped template IDs returning HTTP 200 | 148 |
| Mapped templates with active version | 148 |
| Mapped templates with active HTML | 148 |
| SendGrid account metadata with `SENDGRID_API_KEY` | HTTP 200; paid account; reputation 99 |
| SendGrid event webhook settings with `SENDGRID_API_KEY` | HTTP 200; webhook enabled |
| Sandbox mail-send with `SENDGRID_API_KEY` | HTTP 401; not authorized to send mail |
| Sandbox mail-send with `SENDGRID_API_KEY_FOR_READING` | HTTP 401; not authorized to send mail |

The event webhook is currently enabled for:

```text
https://api.sensorglobal.com/api/v1/admins/settings/email-read-status
```

Enabled event categories include delivered, bounce, deferred, processed, open, click, dropped, spam report, unsubscribe, group unsubscribe, and group resubscribe.

---

## Local Safety Export

Because SendGrid account access may be at risk, a local safety export was captured under `/tmp`:

```text
/tmp/sendgrid-template-export-20260509/
/tmp/sendgrid-template-export-20260509.tar.gz
```

Contents:

- `manifest.csv` with internal template code, SendGrid template ID, HTTP status, active version count, active HTML length, and subject.
- One JSON file per fetched SendGrid template, containing template metadata and version content from the SendGrid API.

The archive is about `72K`; expanded content is about `1.0M`.

This export is local and ephemeral. If it should be retained, move it to approved secure storage. Do not commit the full HTML export to the repo without explicit approval.

---

## Current Backend Template Model

Current production email is SendGrid-centric:

- Internal codes such as `S003`, `S121`, and `S000-P` map to SendGrid dynamic template IDs in `src/config/envWiseTemplateIds.ts/prodTemplateIds.ts`.
- `MailManager.sendMail()` sends `templateId` plus `dynamic_template_data` through `@sendgrid/mail`.
- Local `header.html` and `footer.html` are compiled and injected into `dynamicData.header` and `dynamicData.footer`.
- `tbl_outgoing_communications` stores communication metadata, priority, and cached template content.
- Scheduled email queue creation still fetches template HTML from SendGrid using `SENDGRID_API_KEY_FOR_READING`.
- Email read/open status handling expects SendGrid event payload fields such as `sg_message_id`, `sg_template_id`, and `smtp-id`.

---

## Migration Implications

Moving from SendGrid to SES is not just replacing the send API:

1. **Template source of truth must move.** SES cannot use SendGrid `d-...` dynamic template IDs. The safest path is to render templates in-app from DB/exported HTML, then send finished HTML through SES.
2. **Queued emails must stop fetching template HTML from SendGrid.** Queue creation should use local or DB template content.
3. **Event tracking must be remapped.** SES event payloads differ from SendGrid, so the `/email-read-status` flow needs a new SES-compatible mapper or tracking should be intentionally reduced.
4. **Outbound send needs a new provider layer.** Keep `MailManager` as the call-site API, but add a provider abstraction behind it for SendGrid/SES.
5. **Current SendGrid send capability is questionable.** The configured `SENDGRID_API_KEY` can read account/webhook data but failed sandbox-mode mail-send validation.

---

## Recommended Next Steps

1. Preserve the `/tmp/sendgrid-template-export-20260509.tar.gz` archive in approved secure storage if SendGrid account access may be lost.
2. Create a DB-backed or repo-backed template renderer from the exported SendGrid HTML and current `dynamicData`.
3. Add an email provider abstraction behind `MailManager`.
4. Implement SES sending with rendered HTML, attachments, inline images, and custom message IDs.
5. Add SES event ingestion or explicitly de-scope open/read tracking.
6. Add `EMAIL_PROVIDER=sendgrid|ses` and keep SendGrid as rollback until SES is proven.
7. Validate SES domain identity, DKIM/SPF/DMARC, production send quota, and event destinations before production cutover.
