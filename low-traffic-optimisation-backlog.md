# Low-Traffic Optimisation Backlog

- Created: 2026-04-29
- Purpose: track one-by-one cost and capacity optimisations for the expected low-traffic month.
- Source investigation: `docs/investigations/2026-04-29-low-traffic-mqtt-and-optimisation.md`
- Change log rule: record completed remote changes in `docs/applied-changes.md`, not only here.

## Status Legend

| Status | Meaning |
|---|---|
| `Proposed` | Evidence exists, but the item is not yet ready for approval or execution. |
| `Needs evidence` | More read-only discovery is required before proposing a change. |
| `Needs approval` | Scope is clear and needs explicit human approval before any infra mutation. |
| `Ready to execute externally` | Approved and ready for a human/operator to apply outside this read-only workflow. |
| `Applied externally` | A human/operator reports the remote change was applied. |
| `Verified` | Read-only checks confirmed the applied change. |
| `Deferred` | Deliberately parked for later. |

## Current Focus

`OPT-002 odoo-production io1 to gp3`

## Backlog

### OPT-001 - Sandbox fixed-cost cleanup

- **Status:** `Verified`
- **Priority:** P1
- **Environment:** sandbox
- **Approval reference:** User replied `proceed` on 2026-04-29 after review of `docs/runbooks/10-sandbox-fixed-cost-cleanup.md`; default posture is `Archive then delete`.
- **Evidence:** sandbox EMQX is stopped; sandbox NAT Gateway `nat-0f00d13d55f11c828` had 0 bytes and 0 packets over the 2026-04-22 to 2026-04-29 CloudWatch window; sandbox ALBs `Smoke-Sandbox-Api-LB` and `sandbox-sso-api` had 0 requests; `sensor-sandbox` had 0 DB connections; sandbox Redis memory averaged about 0.5%.
- **Rough saving:** stage 2B is estimated at about `USD 445/month` gross, or about `AUD 630/month`, before taxes and after allowing for only small residual snapshot/backups storage.
- **Recommended action:** if sandbox can stay offline for the quiet month, prepare a snapshot-and-delete or snapshot-and-suspend plan for sandbox fixed-cost resources.
- **Candidate resources:** sandbox NAT Gateway, sandbox ALBs, `sensor-sandbox`, `smokealarmsandbox`, stopped sandbox EC2 instances, attached/unattached sandbox storage, idle sandbox Elastic IPs, and related DNS references.
- **Terraform path:** stage 1, stage 2A, and stage 2B were externally applied and verified. Final read-only sandbox Terraform plan shows `No changes. Your infrastructure matches the configuration.`
- **Approval gate:** confirm whether sandbox must be recoverable quickly, recoverable from backups, or not needed next month.
- **Verification:** re-run read-only NAT, ALB, RDS, ElastiCache, EC2, EBS, EIP, DNS, and Terraform plan checks after each externally applied change.
- **Verified result:** `sensor-sandbox` and `smokealarmsandbox` are not found; NAT Gateway `nat-0f00d13d55f11c828` is `deleted`; NAT EIP `eipalloc-0121eef619a926133` is not found; the five stopped sandbox EC2 instances are terminated; sandbox instance EIPs are released; sandbox API/SSO ALBs are not found.
- **Follow-up:** Route53 sandbox records are outside this Terraform stage and may still point at deleted ALBs or released EIPs.
- **Rollback/recovery:** document snapshots/backups and recreation steps before deletion; for suspended resources, document exact start/resume sequence.
- **Execution packet:** `docs/runbooks/10-sandbox-fixed-cost-cleanup.md`.
- **Source docs:** `docs/investigations/2026-04-29-low-traffic-mqtt-and-optimisation.md`, `docs/investigations/2026-04-20-infra-inventory.md`, `docs/runbooks/08-stop-nonprod-environments.md`.

### OPT-002 - `odoo-production` io1 to gp3

- **Status:** `Needs approval`
- **Priority:** P1
- **Environment:** prod
- **Evidence:** prior RDS deep analysis found `odoo-production` using io1 with 3,000 provisioned IOPS and low IOPS utilisation; current 7-day check showed 6.347% average CPU and stable low connection count.
- **Recommended action:** perform an in-place RDS storage type migration from io1 to gp3 during an approved maintenance window.
- **Candidate resources:** RDS instance `odoo-production`.
- **Approval gate:** production maintenance-window approval and acceptance of the RDS storage optimisation period.
- **Verification:** confirm storage type is gp3, DB is available, Odoo health checks pass, and RDS CPU/connections/IOPS remain normal.
- **Rollback/recovery:** rely on pre-change RDS snapshot and maintenance-window rollback decision; storage type changes may not be immediately reversible during AWS optimisation.
- **Source docs:** `docs/investigations/2026-04-11-rds-deep-analysis.md`, `docs/investigations/2026-04-11-cost-optimization-proposals.md`, `docs/investigations/2026-04-29-low-traffic-mqtt-and-optimisation.md`.

### OPT-003 - Unattached EBS cleanup

- **Status:** `Needs approval`
- **Priority:** P2
- **Environment:** account-wide
- **Evidence:** 24 available/unattached EBS volumes totalling 745 GB were found during the 2026-04-29 read-only inventory.
- **Recommended action:** owner-confirm each unattached volume, snapshot uncertain volumes, then delete confirmed obsolete volumes externally.
- **Candidate resources:** available EBS volumes across prod, sandbox, and decommissioned environments.
- **Approval gate:** volume ownership, retention, and snapshot policy confirmation.
- **Verification:** re-run `describe-volumes` to confirm only approved volumes were removed; verify no unexpected Terraform drift for managed volumes.
- **Rollback/recovery:** restore from snapshot if a deleted volume is later required.
- **Source docs:** `docs/investigations/2026-04-29-low-traffic-mqtt-and-optimisation.md`, `docs/investigations/2026-04-11-cost-optimization-proposals.md`.

### OPT-004 - Idle Elastic IP release

- **Status:** `Needs approval`
- **Priority:** P2
- **Environment:** account-wide
- **Evidence:** 8 unassociated Elastic IPs were found during the 2026-04-29 read-only inventory.
- **Recommended action:** map each EIP to DNS, vendor allowlists, historical docs, and decommissioned resources; release only confirmed obsolete addresses externally.
- **Candidate resources:** unassociated Elastic IP allocations.
- **Approval gate:** DNS, vendor allowlist, and owner confirmation for each address.
- **Verification:** re-run `describe-addresses`; confirm no DNS records, security groups, vendor allowlists, or runbooks still reference released IPs.
- **Rollback/recovery:** no reliable rollback for released public IPs; treat approval as final.
- **Source docs:** `docs/investigations/2026-04-29-low-traffic-mqtt-and-optimisation.md`, `docs/investigations/2026-04-20-infra-inventory.md`.

### OPT-005 - Production EMQX right-size

- **Status:** `Needs evidence`
- **Priority:** P2
- **Environment:** prod
- **Evidence:** prod EMQX receives continuous low-volume traffic, averaging about 1.08 MB/hour inbound, 0.318% CPU, and 5.987% memory on `c5.xlarge`.
- **Recommended action:** collect broker-level client/session/topic metrics before proposing an instance resize.
- **Candidate resources:** EC2 instance `i-0b381a2bfdf65a329` / `Emqx-prod`.
- **Approval gate:** read-only EMQX dashboard or `emqx_ctl` evidence; maintenance-window approval for any resize.
- **Verification:** confirm MQTT client count, topic activity, retained/session counts, EMQX memory, and post-change broker health.
- **Rollback/recovery:** keep AMI/snapshot and instance resize rollback plan; do not stop the broker for cost savings.
- **Source docs:** `docs/investigations/2026-04-29-low-traffic-mqtt-and-optimisation.md`, `docs/infra/prod-services-access.md`, `docs/investigations/2026-04-09-mqtt-device-communications.md`.

### OPT-006 - Redis rightsizing or sandbox Redis removal

- **Status:** `Needs evidence`
- **Priority:** P3
- **Environment:** prod and sandbox
- **Evidence:** prod Redis nodes averaged about 1.5% memory and low CPU; sandbox Redis nodes averaged about 0.5% memory and low CPU.
- **Recommended action:** delete sandbox Redis if sandbox is suspended; consider smaller prod Redis nodes only after confirming data durability expectations and latency sensitivity.
- **Candidate resources:** `smokeprodapiredis`, `smokealarmsandbox`.
- **Approval gate:** confirm whether Redis is purely transient cache/TTL state; confirm failover and snapshot expectations.
- **Verification:** re-run ElastiCache metrics, application health checks, and alarm/notification path checks after any external change.
- **Rollback/recovery:** for sandbox, recreate from Terraform/backups if needed; for prod, keep snapshot and resize rollback plan.
- **Source docs:** `docs/investigations/2026-04-29-low-traffic-mqtt-and-optimisation.md`, `docs/investigations/2026-04-17-prod-snapshot-coverage.md`.

## Notes

- Work these items one by one, starting with `OPT-001`.
- Keep read-only investigation evidence in dated investigation notes.
- Keep actual applied remote changes in `docs/applied-changes.md` with approval reference and verification notes.
