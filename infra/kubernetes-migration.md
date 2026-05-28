# Kubernetes Migration Summary

**Last updated:** 2026-04-29  
**Status:** Research recommendation; no Kubernetes implementation applied

## Recommendation

Use AWS EKS as the target for the first Kubernetes migration phase, and include EMQX in that phase. Keep EMQX as a distinct gated cutover stream because MQTT is production-critical and has more device-side risk than API or SSO traffic.

Phase 1 target:

| Workload | Target |
|---|---|
| `sensor-alarm-backend` | EKS Deployment |
| `sso-provider` | EKS Deployment |
| cron/background execution | EKS CronJobs or worker Deployment after current cron inventory |
| EMQX | EKS-managed EMQX cluster, preferably via EMQX Operator |
| API/SSO ingress | AWS Load Balancer Controller ALB ingress |
| MQTT ingress | AWS Network Load Balancer |

Keep these outside Kubernetes for phase 1:

- RDS MySQL
- ElastiCache Redis
- MongoDB Atlas
- Kafka EC2
- S3/CloudFront Angular portals
- Odoo
- WordPress
- Route 53 and ACM account-level services

## Why EMQX Belongs In Scope

EMQX is currently a single production EC2 instance:

| Item | Value |
|---|---|
| Instance | `i-0b381a2bfdf65a329` |
| Instance type | `c5.xlarge` |
| Private IP | `10.0.2.98` |
| Public IP | `13.211.102.112` |
| Hostname | `mqtt.sensorglobal.com` |
| Version | EMQX `5.8.7` documented |
| Ports | `1883`, `18756`, dashboard `18083` |

Moving it to EKS can improve reproducibility, availability, and platform consistency. It is also the critical path for alarm/event traffic because MQTT alarm events do not flow through Kafka.

The objection is operational risk, not Kubernetes fit. MQTT device clients may depend on broker certificates, source IP, retained messages, sessions, ACLs, and reconnect behavior. These must be proven before cutover.

## Default Design

- Use EMQX Operator for production unless compatibility testing blocks it.
- Use a Kubernetes `Service` of type `LoadBalancer` backed by an AWS Network Load Balancer for MQTT TCP traffic.
- Preserve current MQTT ports and source CIDR restrictions.
- Keep the dashboard private.
- Use persistent volumes for EMQX state if broker discovery confirms retained messages, durable sessions, or on-disk state matter.
- Use External Secrets or Secrets Store CSI Driver backed by AWS Secrets Manager.
- Use IRSA or EKS pod identity for AWS permissions.
- Use ALB ingress only for HTTP/HTTPS workloads such as API and SSO.

Version path:

- Preferred target: EMQX `5.9` or `5.10` with EMQX Operator `2.3.x`, after test-hub validation.
- Parity fallback: EMQX `5.8.7` with EMQX Operator `2.2.x`, then upgrade EMQX in a later phase.

## Required Gates

Do not cut over production until these are complete:

- Current EMQX broker config exported.
- Connected client count, topic activity, retained messages, session state, auth, ACLs, certs, and plugins documented.
- A test hub/SIM validates connection, publish, subscribe, reconnect, and pod restart behavior through the new NLB.
- API pods validate MQTT connectivity to the EKS broker.
- DNS rollback from `mqtt.sensorglobal.com` to the current EC2 broker is documented.
- API and SSO containers expose health/readiness endpoints and run without PM2.
- Terraform/EKS implementation files have explicit approval before editing.

## Risks

- Kubernetes may increase fixed cost if it only hosts a few workloads.
- MQTT cutover can affect field devices and alarm/SMS reliability.
- The current backend uses MQTT QoS `0`, while upstream docs specify QoS `2`; do not change this during the platform migration without a separate protocol decision.
- The app currently starts MQTT/Kafka consumers inside each API process, so replica count changes must be tested carefully.
- EMQX version/operator compatibility must be chosen deliberately because production is documented as `5.8.7`.

## Main Investigation

Detailed plan and evidence: `docs/investigations/2026-04-29-eks-platform-migration-plan.md`.
