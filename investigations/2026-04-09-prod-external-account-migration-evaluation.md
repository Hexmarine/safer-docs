# Production External-Account Migration Evaluation

- Date: 2026-04-09
- Scope: evaluate what a production-first migration from AWS account `747293622182` to a different AWS Organization would involve
- Related systems: Route 53, CloudFront, ACM, WAF, ALB, EC2, RDS, ElastiCache, MQTT/Kafka ingress, IAM, Organizations, KMS, monitoring, messaging, third-party integrations

## Objective

Identify which parts of the current production platform can be rebuilt in a new AWS account, which parts must be revalidated or re-onboarded because they are account-bound, and which shared-account control-plane assets make a production-only move non-trivial.

## Sources Used

- Repository files:
  - `AGENTS.md`
  - `docs/investigations/2026-04-09-aws-account-baseline.md`
  - `docs/investigations/2026-04-09-ap-southeast-2-architecture-baseline.md`
  - `docs/investigations/2026-04-09-platform-architecture-and-data-flow.md`
  - `docs/investigations/2026-04-09-code-component-review.md`
  - `docs/investigations/2026-04-09-cloudwatch-newrelic-deep-dive.md`
  - `docs/investigations/2026-04-09-vanta-recommendation.md`
- AWS commands or consoles:
  - `aws organizations describe-organization`
  - `aws route53 list-hosted-zones`
  - `aws route53 list-resource-record-sets --hosted-zone-id Z03520551S3PY3J02L5ZN`
  - `aws acm list-certificates --region ap-southeast-2 --certificate-statuses ISSUED`
  - `aws acm list-certificates --region us-east-1 --certificate-statuses ISSUED`
  - `aws acm describe-certificate --region us-east-1 --certificate-arn arn:aws:acm:us-east-1:747293622182:certificate/dbd399a5-1e3f-48ed-83aa-f6250f7a3adc`
  - `aws wafv2 list-web-acls --scope REGIONAL --region ap-southeast-2`
  - `aws wafv2 list-web-acls --scope CLOUDFRONT --region us-east-1`
  - `aws cloudfront list-distributions`
  - `aws cloudfront list-functions --stage LIVE`
  - `aws globalaccelerator list-accelerators --region us-west-2`
  - `aws kms list-aliases --region ap-southeast-2`
- External references: none

## Findings

### Executive Summary

- A production move to a different AWS Organization is not just an EC2/RDS relocation.
- The hardest parts are account-bound control-plane assets:
  - Route 53 hosted zones and production DNS cutover
  - CloudFront distributions and their shared `us-east-1` certificate
  - WAF and security-control recreation
  - IAM, KMS, CloudTrail/Security Hub/Config style governance reset in a new organization
  - third-party integrations that are tied to this account or its DNS
- The production runtime can be rebuilt, but the move is gated by public edge, certificates, DNS ownership, stateful data migration, and re-onboarding of monitoring and messaging integrations.

### Current Migration Shape

- The current source account is an AWS Organizations member account under organization `o-l2ps3sa79r`.
- The Organizations management account is `872515272604`.
- SCPs are enabled, so an external destination account will not inherit the current org guardrails automatically.
- The account owns public DNS directly:
  - `14` Route 53 hosted zones were observed
  - `sensorglobal.com.` alone contains `119` record sets
- Production traffic is split across several ingress styles:
  - CloudFront for the main Angular-style portal hostnames
  - ALBs for API, SSO, keychain, and WordPress
  - direct `A` records for some production endpoints such as `sensorglobal.com.`, `mqtt.sensorglobal.com.`, `kafka.sensorglobal.com.`, and `monitoring.sensorglobal.com.`
- This means the migration must handle both managed front doors and direct instance-style endpoints.

### Production Public Entry Points Observed

- CloudFront-backed production portals:
  - `admin.sensorglobal.com`
  - `agent.sensorglobal.com`
  - `trader.sensorglobal.com`
  - `activation.sensorglobal.com`
  - `reschedule.sensorglobal.com`
  - `owner.sensorglobal.com`
- ALB-backed production hostnames:
  - `api.sensorglobal.com`
  - `auth.sensorglobal.com`
  - `keychain.sensorglobal.com`
  - `web.sensorglobal.com`
- Direct-address production hostnames:
  - `sensorglobal.com`
  - `mqtt.sensorglobal.com`
  - `kafka.sensorglobal.com`
  - `monitoring.sensorglobal.com`

### Certificate And Edge Constraints

- Five CloudFront distributions with environment aliases share the same imported `us-east-1` certificate:
  - `arn:aws:acm:us-east-1:747293622182:certificate/dbd399a5-1e3f-48ed-83aa-f6250f7a3adc`
- That certificate covers:
  - `*.sensorglobal.com`
  - `sensorglobal.com`
- Its status is:
  - `IMPORTED`
  - `EXPIRED`
  - `RenewalEligibility = INELIGIBLE`
  - `NotAfter = 2026-04-09T09:59:59+10:00`
- This is a critical migration blocker and likely an operational issue even before migration. A new external account cannot rely on copying this certificate and would need a fresh certificate issuance or import plus DNS validation.
- Regional ACM certificates also exist in `ap-southeast-2` for:
  - `*.sensorglobal.com`
  - `sensorinsure.com` and `sensorinsure.com.au`
- Regional WAF ACLs exist for:
  - production core web
  - production keychain
  - QA, sandbox, and dev environments
- CloudFront-scope WAF ACLs also exist, but current CloudFront distribution summaries showed blank `WebACLId` values for the aliased distributions, so the production attachment state still needs confirmation.
- One Global Accelerator exists, but it appears development-focused:
  - `SmokedevelopmentLBAccelerator`
- That accelerator is not currently a production blocker for a prod-first migration, but it does show the account already uses global-entry-point features in at least one environment.

### Shared Assets That Complicate A Prod-Only Move

- The `keychain` DNS pattern appears shared across environments:
  - `keychain-development.sensorglobal.com`
  - `keychain-qa.sensorglobal.com`
  - `keychain-sandbox.sensorglobal.com`
  - `keychain.sensorglobal.com`
  - all currently point to the same production keychain ALB
- That means moving only production keychain may break non-prod hostnames unless those shared DNS patterns are redesigned first.
- The wildcard `*.sensorglobal.com` certificate is also shared across prod and non-prod CloudFront aliases.
- The `sensorglobal.com` hosted zone contains non-production and external-verification records mixed with production records, so zone ownership and record cutover cannot be treated as production-only in a purely clean way.

### Migration Dependency Matrix

| Domain or system | Current asset | Migration type | Account-bound reason | Cutover sensitivity | Likely owner | Blocking unknowns |
| --- | --- | --- | --- | --- | --- | --- |
| Public DNS and registrar control | Route 53 hosted zones, especially `sensorglobal.com` with `119` records | Recreate zone in target or retain DNS in source and do staged record cutover | Hosted zone ownership, registrar-created zones, and validation records live in the source account | very high | platform / DNS | whether domains will stay registered in the source account or move too |
| CloudFront production portals | Production portal distribution `E1UDNFPYDRUZOD` plus related alias pattern | Rebuild distribution and re-point aliases | Distribution IDs, alternate domain ownership, and edge config are account-local | very high | platform / frontend | exact origin buckets and deployment path for prod frontend assets |
| CloudFront certificate | Imported wildcard cert `dbd399a5-...` in `us-east-1` | Replace, re-issue, or re-import in target account | Imported ACM certs are not portable and this one is expired/ineligible | very high | platform / security | whether private key material still exists outside AWS for re-import, or if fresh issuance is required |
| Regional ALB ingress | `api.sensorglobal.com`, `auth.sensorglobal.com`, `keychain.sensorglobal.com`, `web.sensorglobal.com` | Rebuild ALBs, target groups, listeners, SGs, and DNS targets | ALBs, target groups, SGs, and listener bindings are account-local | high | platform | current listener, target-group, and health-check mapping per hostname |
| WAF controls | `Sensor_Global_Prod_web_ACL`, `Sensor_Global_Prod_key-chain_web_ACL` | Recreate rules and reassociate in target | WAF ACLs are account-local resources | high | security / platform | full rule contents and any logging destinations not yet captured |
| Core application compute | production EC2 API, SSO, keychain, WordPress, Odoo, support hosts | Rebuild and redeploy | EC2 instances, IAM instance profiles, SGs, EIPs, and userdata are account-local | medium to high | platform / application | exact deployment automation and AMI/bootstrap assumptions |
| Stateful relational data | `sensor-prod` MySQL, `odoo-production` PostgreSQL | Snapshot/restore, dump/restore, or replication-based migration | RDS snapshots and endpoints are account-scoped and application cutover must preserve data integrity | very high | platform / DBA / app | downtime tolerance, replication feasibility, and actual production data volume |
| Stateful cache/data-adjacent services | production Redis and supporting cache footprint | Rebuild or migrate warm state with planned cache invalidation | ElastiCache is account-local and state is not just copied over | medium | platform / app | whether cache can be cold-started safely or needs warm migration |
| MQTT and Kafka ingress | `mqtt.sensorglobal.com = 13.211.102.112`, `kafka.sensorglobal.com = 13.237.253.114` | Rebuild broker hosts and do staged DNS/client cutover | Direct `A` records point to account-local public IPs and instances | very high | platform / IoT | client reconnect behavior, certificate/auth setup, and whether devices pin endpoints or trust roots |
| Root site and direct-IP endpoints | `sensorglobal.com = 54.253.158.197`, `monitoring.sensorglobal.com = 52.64.196.201` | Replace direct targets with rebuilt services and new DNS | Direct A records bypass managed abstraction and tie cutover to single hosts | very high | platform / webops | what application actually serves the root domain today and whether monitoring is business-critical during cutover |
| Organization and security baseline | member-account status under org `o-l2ps3sa79r`, SCP-enabled posture | Redesign and recreate in destination org/account | org membership, SCPs, delegated admin, and account-level baselines do not transfer to a different organization | high | security / cloud governance | target org design, required guardrails, and delegated security service model |
| KMS and encrypted security tooling | customer-managed aliases such as `alias/cloudtrail-key`, `alias/security-hub`, `alias/export-keys`, `alias/sh_export_key`, `alias/temp-key` | Recreate keys and repoint producers/consumers | KMS keys are account-bound and key IDs/ARNs change | high | security / platform | which application or logging systems depend on each custom key |
| Monitoring export | `NewRelic-Metric-Stream` plus `NewRelic-Delivery-Stream` | Recreate integration and re-authorize destination account | Metric streams, Firehose, and third-party account trust are account-local | medium | platform / observability | exact New Relic account ownership and namespace requirements |
| Messaging and notification integrations | AWS End User Messaging setup, Twilio, SendGrid, email/SMS DNS validation records | Re-onboard and revalidate where account identity matters | sender identity, webhook allow-lists, DNS validation, and account trust may need re-registration | high | product / platform | which channels are AWS-native versus third-party and what legal sender registrations exist |
| SaaS and business integrations | PropertyMe, Odoo workflows, Microsoft 365 validation records, KORE SIM operational systems | Repoint credentials, webhooks, allow-lists, and trust boundaries | integration credentials and callback origins are often account- or domain-specific | medium to high | application / operations | which integrations enforce source IP, hostname, or account-level trust |
| Operational access and compliance tooling | OpenVPN, Vanta, Config, Security Hub, Inspector | Rebuild or re-enroll as needed | marketplace licensing, auditor roles, and compliance enrollments are account-specific | medium | security / operations | which of these are mandatory on day one of migration versus follow-on hardening |

### What Looks Portable Versus What Does Not

- Most portable by rebuild:
  - stateless application EC2 workloads
  - ALB layer
  - frontend artifact hosting if deployment pipelines and buckets are recreated
- Least portable:
  - DNS ownership and registrar-linked hosted zones
  - imported CloudFront certificate
  - WAF, IAM, KMS, and org/governance configuration
  - stateful RDS and Redis data plane
  - direct-IP MQTT/Kafka endpoints used by external clients or devices
  - third-party integrations that trust this account, these hostnames, or these validation records

## Risks Or Constraints

- The production migration surface is larger than a normal “web app plus database” move because the account also owns DNS, certificates, messaging, device ingress, and security tooling.
- A prod-only move is complicated by shared assets, especially:
  - the wildcard `sensorglobal.com` certificate
  - shared Route 53 zone ownership
  - `keychain` hostnames from non-prod pointing at the prod ALB
- Direct `A` record endpoints such as `sensorglobal.com`, `mqtt.sensorglobal.com`, and `kafka.sensorglobal.com` reduce the abstraction available during cutover and increase rollback sensitivity.
- The expired imported CloudFront certificate is both a migration risk and a current-state operational risk.
- Because the destination is outside the current AWS Organization, governance, SCPs, delegated security services, and audit tooling all need to be intentionally re-established.

## Cost Optimization Opportunities

- Cost reduction is not the primary purpose of this note.
- A migration program may create an opportunity to remove duplicated or legacy non-prod assets, but those savings should be treated as secondary to safe cutover.
- The clearest accidental cost opportunity is to avoid migrating non-essential shared or duplicated non-prod dependencies into the new production account.

## Recommended Next Steps

- Resolve the certificate issue first:
  - confirm whether the private key for the imported `*.sensorglobal.com` certificate still exists
  - if not, plan fresh issuance and DNS validation in the destination account
- Build a production DNS cutover inventory from the `sensorglobal.com` zone with explicit ownership for:
  - CloudFront aliases
  - ALB-backed records
  - direct `A` records
  - external validation and email records that must survive the move
- Confirm whether production will:
  - keep DNS hosted in the source account during migration
  - move hosted zones and registrar control to the destination account
- Create a dedicated data-migration investigation for:
  - `sensor-prod`
  - `odoo-production`
  - production Redis / cache dependencies
- Create a dedicated IoT ingress investigation for:
  - `mqtt.sensorglobal.com`
  - `kafka.sensorglobal.com`
  - any certificate, auth, or reconnect behavior tied to current broker hosts
- Document the target-account baseline requirements:
  - IAM strategy
  - SCP/governance expectations
  - KMS layout
  - CloudTrail / Config / Security Hub / Inspector posture
  - third-party re-onboarding checklist

## Open Questions

- Does the organization intend to move domain registration and hosted zones, or only the workloads behind existing DNS names?
- Is the expired imported CloudFront certificate still backed by recoverable private key material outside AWS?
- What currently serves `sensorglobal.com` directly, and is that root-domain behavior intentionally production-critical?
- Do device or partner clients pin any broker certificate, hostname, or IP assumptions for MQTT or Kafka connectivity?
- Which external integrations require callback URL, source-IP, or allow-list changes before a destination account can go live?
- Is there a target external AWS account already available, or does the destination governance model still need to be designed?
