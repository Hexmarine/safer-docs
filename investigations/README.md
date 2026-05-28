# Investigations index

Dated discovery notes for the SensorGlobal → Safer Homes platform handover:
architecture mapping, cost/security analysis, incident root-causes, and migration
planning. Each file is a **point-in-time record** — kept as history even once the
finding is acted on. Newer notes supersede older ones on the same topic; the
**Latest / authoritative** column below points to the one to read first.

Stable, continuously-current knowledge lives in [`../infra/`](../infra/) (topic
docs) and [`../runbooks/`](../runbooks/) (executable procedures); investigation
notes link out to those when a finding has graduated into one.

Naming: `YYYY-MM-DD-short-topic.md`.

---

## AWS cost & optimisation
The April notes are a progression (quick-wins → deep-dive → proposals); read the
proposals doc for the actionable, costed list, the others for how it was derived.

| Note | Covers |
|---|---|
| `2026-04-09-cost-quick-wins.md` | First-pass quick wins: CloudWatch, NAT, RDS, ElastiCache, SMS, OpenVPN |
| `2026-04-10-cost-deep-dive.md` | Feb–Apr baseline refresh; EBS/PIOPS/SMS/ElastiCache breakdown |
| **`2026-04-11-cost-optimization-proposals.md`** | **Authoritative** costed proposals with savings, risk, compliance constraints → runbooks `01`–`05`, `10` |
| `2026-04-09-cloudwatch-newrelic-deep-dive.md` | CloudWatch cost drivers + New Relic metric-stream impact |
| `2026-04-11-rds-deep-analysis.md` | RDS load/IOPS analysis behind the io1→gp3 call |
| `2026-05-06-rds-cache-cost-optimisation.md` | RDS io1→gp3 + gp2→gp3 safety; ElastiCache reserved-instance review |

## Infrastructure baseline & architecture
| Note | Covers |
|---|---|
| `2026-04-09-aws-account-baseline.md` | Account validation, region footprint, IAM summary |
| `2026-04-09-ap-southeast-2-architecture-baseline.md` | VPC/subnet/NAT/ALB/EC2/RDS layout → graduated to `infra/ap-southeast-2-architecture.md` |
| `2026-04-09-platform-architecture-and-data-flow.md` | End-to-end: device→hub→MQTT→backend→MySQL/MongoDB→portals |
| `2026-04-09-code-component-review.md` | Repo classification: core / support / prototype / incomplete |
| `2026-04-12-upstream-tech-docs-review.md` | Cross-reference of the 12 upstream **Technical Documents** PDFs; confirmations & contradictions |
| `2026-05-29-upstream-platform-docs-extraction.md` | The other 7 upstream docs (Tech Brief, System Description, Architecture, Ubiquitous Language, Security Overview, DevOps, Company Tools) + Installation Process; searchable text in `../upstream/extracted/`; staleness flags + the A004/User-Manuals lead (#102) |
| `2026-05-15-backend-monolith-architecture.md` | Backend is one deployment: API + MQTT worker + Kafka consumer + Socket.IO |
| `2026-04-20-infra-inventory.md` | ⚠️ Superseded — all-services cost/footprint snapshot; current state in `infra/ap-southeast-2-architecture.md` |

## MQTT & device communications
| Note | Covers |
|---|---|
| **`2026-04-09-mqtt-device-communications.md`** | **Authoritative** MQTT topic structure, hub-centric design, auth, provisioning |
| `2026-04-09-alarm-event-traces.md` | Backend event flow: hub event → dispatch → state → log → notify |
| `2026-04-10-inv-results.md` | EMQX 5.8.7 single-node; port 18756 hub TLS; cert expiry finding |
| `2026-05-15-mqtt-lab-routing-design.md` | Safe prod→QA/UAT MQTT mirroring without moving the field fleet |
| `2026-04-29-low-traffic-mqtt-and-optimisation.md` | Quiet-month MQTT activity check + cost candidates |

## Auth, SSO & sessions
| Note | Covers |
|---|---|
| **`2026-04-12-auth-architecture.md`** | **Authoritative** portal OIDC/RS256 vs mobile API sessions vs M2M |
| `2026-05-08-prod-login-sso-token-500.md` | Post-ASG login failure; SSO instance / terms-lookup analysis |
| `2026-05-02-subadmin-self-hidden.md` | Sub-admin absent from own listing (deliberate, not a bug) |
| `2026-05-02-api-playground.md` | Safe repeatable read-only API call pattern |

## Incidents & active issues
Incident records are kept as postmortem history even after resolution.

| Note | Covers |
|---|---|
| `2026-04-11-sms-drop-investigation.md` | SMS sends down 93% since 2026-03-26 (smoke-alarm safety) — discovery |
| `2026-04-12-mysql-sms-root-cause.md` | Root cause: `tbl_alarm_alerts` stopped populating mid-day 2026-03-26 |
| `2026-04-12-certificate-expiry-incident.md` | DigiCert wildcard expired 2026-04-09; HTTPS broke across web apps (resolved) |
| `2026-04-12-current-env-maintenance-actions.md` | Live P0/P1 action tracker: cert, SMS, EMQX TLS, snapshots, DLM |
| `2026-05-08-aws-admin-access-and-asg-scale-down.md` | ASG zeroed by a OneLogin federated user; IAM access analysis → runbook `12` |
| `2026-05-08-prod-sms-global-disable.md` | Emergency SMS gate `SEND_SMS=0` in `sensor-prod` |
| `2026-05-09-sendgrid-template-migration.md` | SendGrid: read key OK, send-mail key unauthorised |

## Migration, account detach & EKS
| Note | Covers |
|---|---|
| **`2026-04-10-migration-plan.md`** | **Authoritative** phased migration plan (pre-flight → phases 1–7); KORE re-reg risk |
| `2026-04-09-prod-external-account-migration-evaluation.md` | Route53/CloudFront/ACM/WAF/IAM account-bound constraints |
| `2026-04-10-prod-user-device-migration-readiness.md` | Live users/devices confirmed; 16 hub EMQX clients active |
| `2026-04-28-aws-account-detach-analysis.md` | Detach from parent org: billing/governance/legal cutover → runbook `09` |
| `2026-04-29-eks-platform-migration-plan.md` | EKS phase 1: backend/SSO/cron/EMQX on K8s; RDS/S3/Odoo stay EC2 |
| `2026-04-29-terraform-drift-status.md` | Terraform baseline reconciliation (prod clean; non-prod deactivated) |

## Deployment, pipelines & local dev
| Note | Covers |
|---|---|
| `2026-05-09-backend-deployment-pipelines.md` | CodePipeline→CodeBuild→CodeDeploy Blue/Green → `infra/deployment-pipeline.md` |
| `2026-05-09-backend-pipeline-validation-and-versioning.md` | Docs-only commit validation; git branching patterns |
| `2026-04-12-kafka-message-flow.md` | Kafka single `logs-production` topic; 30+ producers; smoke-alarm consumer |
| `2026-04-29-local-backend-integration-prep.md` | Local backend validation: `NODE_ENV=local`, MySQL/Mongo/Redis fixtures |

## Vendors & integrations
| Note | Covers |
|---|---|
| **`2026-04-21-odoo-integrations-analysis.md`** | **Authoritative** Odoo 16 + 14 addons; PropertyMe/Shippit/CBA/SendGrid/SES |
| `2026-05-11-odoo-email-sendgrid-ses.md` | Odoo Postfix→SendGrid SMTP, separate from backend SES cutover |
| `2026-05-02-sim-activation-flow.md` | KORE/Twilio Super SIM lifecycle; factory vs runtime activation |
| `2026-04-09-vanta-research.md` | What Vanta is; pricing; active IAM footprint |
| `2026-04-09-vanta-recommendation.md` | Recommend renegotiate & right-size → runbook `06` |

## Backup, DR & data custody
| Note | Covers |
|---|---|
| `2026-04-17-prod-snapshot-coverage.md` | MongoDB on Atlas (not EC2); RDS 7-day backups; EBS snapshot gaps |
| `2026-05-01-mongo-custody-and-aws-restore.md` | Atlas ownership uncertainty; DocumentDB/mongodump recovery → runbook `11` |

## Environment discovery & process
| Note | Covers |
|---|---|
| `2026-04-16-uat-environment-discovery.md` | ⚠️ Superseded — UAT inventory; decommissioned 2026-04-21 (runbook `08`) |
| `2026-04-09-repo-governance-bootstrap.md` | Repo guardrails, investigation structure, approval policy |
| `2026-05-03-emqx-access-and-dashboard-credentials.md` | EMQX prod access via SSM; client census; dashboard recovery |

## Application & install flow
| Note | Covers |
|---|---|
| `2026-05-01-mobile-app-builds.md` | Android Kotlin + iOS native; Gradle flavors; Firebase/AppAuth |
| `2026-05-21-canonical-install-flow-swimlanes.md` | Original on-site native-app install flow timeline |
| `2026-05-20-installation-flow-durability-plan.md` | New depot-kit install + recovery ladder — feeds the `safer-ops` work |

> The depot prepared-kit build that grew out of the last two notes lives in its own
> repo: [`../../code/safer-ops/docs/`](../../code/safer-ops/docs/README.md).

---

## Generated artifacts
Non-markdown companions live in [`artifacts/`](artifacts/):
`2026-04-20-infra-inventory.xlsx` and `2026-04-20-infra-report-overview.html`
(the spreadsheet/HTML renders of `2026-04-20-infra-inventory.md`).
