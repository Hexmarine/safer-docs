# Production SMS Global Disable Plan

**Date:** 2026-05-08  
**Account:** `747293622182` (`ap-southeast-2`)  
**Environment:** Production backend API (`sensor-alarm-backend`)  
**Status:** Planned; production change not yet recorded as applied  
**Classification:** Temporary operational pause

---

## Summary

Production backend SMS has one confirmed global application gate:

```text
sensor-prod.SEND_SMS
```

Setting `SEND_SMS=0` in AWS Secrets Manager secret `sensor-prod` is the selected temporary control for fully disabling backend SMS. The running backend loads `sensor-prod` at process startup, so production API processes must be restarted after the secret value changes.

This is a full SMS pause. It disables OTPs and non-OTP SMS notifications together.

---

## Current Backend SMS Gate

The backend SMS helper checks:

```text
process.env.SEND_SMS == '1'
```

If this is not true, the helper does not publish the SMS through AWS SNS.

There is a second allow-list mode:

```text
process.env.SEND_SMS_ONLY_ON_ALLOWED_LIST == '1'
```

That mode is recipient-based, not message-type-based. It only allows SMS to active numbers in `tbl_sms_allowed_list`; it does not distinguish OTPs from alarm, job, landlord, tenant, or queued SMS.

No existing config gate was found for:

- OTP-only SMS
- non-OTP-only SMS
- alarm-only SMS
- job-only SMS
- landlord/tenant-only SMS

---

## Impact

When `sensor-prod.SEND_SMS=0`, the backend SMS helper blocks all backend-driven SMS categories that use it:

- OTP and account verification SMS
- alarm and device-event SMS
- job assignment, unassignment, scheduling, and service-staff SMS
- tenant entry-notice SMS
- landlord and asset-owner invitations, access links, and reminders
- test alarm SMS
- queued/scheduled SMS sent through the backend queue worker

Email, push notifications, MQTT ingestion, API behavior, and in-app workflows are not intentionally disabled by this setting.

---

## Production Execution Plan

This repository is investigation-first and read-only for production infrastructure. Codex should not mutate Secrets Manager or restart production services directly.

The approved external operator flow is:

1. Capture a pre-change health baseline.
2. Confirm current production secret values:
   - `sensor-prod.SEND_SMS=1`
   - `sensor-prod.SEND_SMS_ONLY_ON_ALLOWED_LIST=0`
3. Update only `sensor-prod.SEND_SMS`:
   - from `1`
   - to `0`
4. Restart production backend API PM2 processes on the active prod API instances so the processes reload `sensor-prod`.
5. Run post-change health checks.
6. Record the actual applied change in `docs/applied-changes.md` only after the production change is complete and verified.

---

## Verification

Post-change verification should confirm:

- `sensor-prod.SEND_SMS=0` by read-only Secrets Manager check.
- Production API processes restarted after the secret update.
- ALB healthy hosts remain healthy.
- API health check remains healthy.
- No fresh backend startup errors appear in logs.
- AWS SMS delivery metrics decline as expected after the approved pause window.

The existing health-check script treats low SMS volume as a safety warning. During this approved temporary pause, that warning is expected, but the rest of the health signals should still be monitored normally.

---

## Rollback

To re-enable backend SMS:

1. Update `sensor-prod.SEND_SMS` back to `1`.
2. Restart production backend API PM2 processes so they reload the secret.
3. Confirm API health.
4. Confirm SMS delivery resumes through AWS SMS metrics or a controlled application-level test.

---

## Follow-Up Question

The current backend does not have a selective gate for non-OTP SMS. If the business wants to pause alarm/job/landlord/tenant notifications while keeping OTPs active, a small backend change is required.

The likely follow-up design should introduce a separate non-OTP or template/category-aware SMS gate while preserving OTP delivery.
