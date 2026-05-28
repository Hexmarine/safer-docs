# EKS Platform Migration Plan

- Date: 2026-04-29
- Scope: research plan for moving the first production platform slice to AWS EKS, including the app tier and EMQX MQTT broker
- Related systems: sensor-alarm-backend, sso-provider, cron/background workers, EMQX, ALB/NLB ingress, Route 53, ACM, Secrets Manager, RDS MySQL, ElastiCache Redis, MongoDB Atlas, Kafka, S3/CloudFront portals, Terraform

## Objective

Define a safe first-phase Kubernetes migration target for SensorSyn. The default target is AWS EKS in `ap-southeast-2`, with API, SSO, background execution, and EMQX included in phase 1. Databases, static portals, Kafka, Odoo, WordPress, and external SaaS dependencies stay outside Kubernetes for this first phase unless later evidence creates a stronger reason to move them.

This is a planning and research artifact only. No AWS infrastructure changes were made.

## Sources Used

- Repository files:
  - `docs/README.md`
  - `docs/infra/ap-southeast-2-architecture.md`
  - `docs/infra/deployment-pipeline.md`
  - `docs/infra/infrastructure-provisioning.md`
  - `docs/infra/prod-services-access.md`
  - `docs/investigations/2026-04-09-platform-architecture-and-data-flow.md`
  - `docs/investigations/2026-04-09-mqtt-device-communications.md`
  - `docs/investigations/2026-04-12-kafka-message-flow.md`
  - `docs/investigations/2026-04-20-infra-inventory.md`
  - `docs/investigations/2026-04-29-low-traffic-mqtt-and-optimisation.md`
  - `docs/investigations/2026-04-29-terraform-drift-status.md`
  - `code/sensor-alarm-backend/package.json`
  - `code/sensor-alarm-backend/appspec.yml`
  - `code/sensor-alarm-backend/src/index.ts`
  - `code/sensor-alarm-backend/src/utils/BootStrap.ts`
  - `code/sensor-alarm-backend/src/services/mqtt/subscriber.ts`
  - `code/sensor-alarm-backend/src/services/kafka/consumer.ts`
  - `code/sso-provider/package.json`
  - `code/sso-provider/appspec.yml`
  - `code/sso-provider/src/index.ts`
  - `code/sso-provider/src/app.ts`
- AWS read-only commands:
  - `aws sts get-caller-identity`
  - `aws ec2 describe-instances --filters Name=tag:Name,Values=Emqx-prod`
  - `aws ec2 describe-addresses --public-ips 13.211.102.112`
- External references:
  - EMQX: Deploy EMQX in Kubernetes, https://docs.emqx.com/en/emqx/latest/deploy/kubernetes/kubernetes.html
  - EMQX: Operator overview and compatibility, https://docs.emqx.com/en/emqx/latest/deploy/kubernetes/operator/operator.html
  - EMQX: Helm chart deployment parameters, https://docs.emqx.com/en/emqx/latest/deploy/kubernetes/chart.html
  - AWS: Route TCP and UDP traffic with Network Load Balancers, https://docs.aws.amazon.com/eks/latest/userguide/network-load-balancing.html

## Findings

### Current production shape

The production app tier is not containerized today. The main API is a Node/Express TypeScript service deployed through GitHub, CodePipeline, CodeBuild, CodeDeploy, PM2, and an Auto Scaling Group behind `SmokeAPI-LB`. The SSO service is a separate Koa/OIDC TypeScript service deployed with the same EC2/PM2 pattern behind `sso-api-lb`.

The main API starts several responsibilities inside one process:

- HTTP API and Swagger routes.
- MySQL connection via Sequelize.
- MongoDB archive and primary connections via Mongoose.
- MQTT subscriber to `$share/sensorGroup/sg/sas/resp/+`.
- Kafka producer and consumer for `logs-production`.
- Socket.IO services.

The current API code does not expose a clearly dedicated health or readiness endpoint. Kubernetes readiness should not be based only on the root route because the root route is Swagger-gated and may return `401` depending on `ENABLE_SWAGGER`.

Static Angular portals are already an object-storage/CDN workload through S3 and CloudFront. They should not move to Kubernetes in phase 1.

Kafka is present, but it is not on the alarm event path. Prior investigation found MQTT alarm events are handled directly in the API MQTT subscriber and written to MySQL/MongoDB. Kafka is used mainly as an async audit/task queue. This makes EMQX more migration-critical than Kafka for phase 1.

### Current EMQX shape

Live read-only AWS discovery and existing docs agree that production EMQX is currently a single EC2 instance:

| Item | Value |
|---|---|
| Instance | `i-0b381a2bfdf65a329` |
| Name | `Emqx-prod` |
| Instance type | `c5.xlarge` |
| State | `running` |
| VPC | `vpc-03ec09702866ca7b1` |
| Subnet | `subnet-09505e030d7cdbacc` |
| Private IP | `10.0.2.98` |
| Public IP | `13.211.102.112` |
| EIP allocation | `eipalloc-09901210d3bf97185` |
| Security group | `sg-03a5a13ee645ca05a` |
| AMI | `ami-0d02292614a3b0df1` |
| Public DNS | `mqtt.sensorglobal.com` |
| Documented version | EMQX `5.8.7` |
| Documented ports | `1883`, `18756`, dashboard `18083` |

The EIP is tagged `Hive-MQ-Prod`, which is a historical naming mismatch. The active service is documented as EMQX.

Production EMQX showed continuous low-volume traffic in the recent optimisation check. Host-level CloudWatch metrics showed low CPU and memory, but exact client counts, topic activity, retained messages, session state, ACLs, and certificate configuration remain unknown because broker-level evidence was not collected.

### EMQX on Kubernetes fit

Moving EMQX to EKS makes architectural sense if the goal is platform consolidation, repeatable deployment, and better HA. EMQX has first-party Kubernetes support through both the EMQX Operator and Helm chart. EMQX documentation recommends the Operator for production and serious pre-production use because it automates lifecycle tasks such as scaling, upgrades, failure recovery, and workload migration.

The version decision matters:

- Current production is documented as EMQX `5.8.7`.
- EMQX Operator `2.2.x` is compatible with EMQX `5.1.1` through `5.8.x`.
- EMQX Operator `2.3.x` is fully compatible with EMQX `5.9`, `5.10`, and `6.0+`.

Default implementation recommendation: use the EMQX Operator for the target architecture, but run an explicit compatibility gate before choosing the final version path. Prefer upgrading EMQX to a `5.9` or `5.10` target with Operator `2.3.x` if a test hub and rollback path are available. If version parity is more important for the first cutover, pin to the Operator `2.2.x` line for EMQX `5.8.7` and plan the broker upgrade as a follow-up.

### Target phase 1 workload map

| Current system | Phase 1 target | Notes |
|---|---|---|
| `sensor-alarm-backend` EC2/PM2/ASG | EKS Deployment | Requires container image, health endpoint, graceful shutdown, stdout logs, Secrets Manager integration, and scaling policy. |
| API MQTT/Kafka/socket responsibilities | Same API pod initially | Do not split responsibilities until behavior is proven. Add worker split only if singleton or scaling issues are confirmed. |
| `sso-provider` EC2/PM2 | EKS Deployment | Requires issuer/callback/cookie validation behind ingress. |
| Cron server/background jobs | EKS CronJobs or worker Deployment | First inventory current cron tasks from EC2 before deciding exact Kubernetes shape. |
| EMQX EC2 | EKS EMQX cluster | Use EMQX Operator by default; expose MQTT via Network Load Balancer. |
| API/SSO ALBs | AWS Load Balancer Controller ALB Ingress | Use ALB for HTTP/HTTPS only. |
| MQTT direct public endpoint | EKS Service type `LoadBalancer` backed by NLB | Use NLB for TCP MQTT. Preserve source ranges and TLS/client behavior. |
| Secrets Manager secret `sensor-prod` | External Secrets or Secrets Store CSI Driver | Do not bake secrets into images or ConfigMaps. |
| RDS MySQL, Redis, MongoDB Atlas | External dependencies | Keep outside Kubernetes in phase 1. |
| Kafka EC2 | External dependency | Keep outside Kubernetes in phase 1. |
| Angular portals | S3/CloudFront | Keep outside Kubernetes. |
| Odoo/WordPress | EC2/RDS | Keep outside Kubernetes. |

## Recommended Target Architecture

### Cluster and namespaces

Create an EKS cluster in `ap-southeast-2`, ideally in the production VPC or a peered/VPC-routed environment that can reach RDS, Redis, EC2 Kafka, and any required internal services without exposing databases publicly.

Use separate namespaces at minimum:

- `sensorsyn-prod` for API, SSO, and workers.
- `mqtt-prod` for EMQX.
- `platform-system` or equivalent for controllers such as AWS Load Balancer Controller, External Secrets, cert-manager if used, and observability components.

### Ingress

Use ALB ingress for HTTP services:

- `api.sensorglobal.com`
- `auth.sensorglobal.com`
- any temporary migration hostnames such as `api-k8s.sensorglobal.com`

Use NLB for MQTT TCP services:

- `mqtt.sensorglobal.com`
- temporary validation hostname such as `mqtt-k8s.sensorglobal.com`

AWS EKS documentation recommends the AWS Load Balancer Controller for NLB provisioning, with Kubernetes `Service` resources of type `LoadBalancer`. Use IP targets if the chosen CNI and node model support it. Preserve the existing MQTT source restrictions from current security groups, especially KORE/ONO SIM CIDRs and any backend/VPC ranges.

If retaining the exact public IP matters, research assigning Elastic IP allocations to the NLB. Do not assume the current EC2 EIP can be reused without a cutover design and rollback decision.

### Secrets and identity

Use pod identity/IRSA for AWS API access. Avoid static AWS keys in pod environment variables.

Use Secrets Manager as the source of truth for sensitive application values. The Kubernetes layer should reference secrets through External Secrets Operator or Secrets Store CSI Driver. Do not retrieve or copy secret values during planning unless explicitly approved.

### EMQX target

Deploy EMQX as a Kubernetes-managed stateful service:

- EMQX Operator preferred for production.
- StatefulSet/PVC-backed storage through the operator or chart.
- Pod disruption budget.
- Multi-AZ node placement.
- Dedicated node pool or taints/tolerations if MQTT latency or noisy-neighbor risk appears material.
- NLB TCP listener for MQTT ports.
- Dashboard kept private, reachable only through a controlled tunnel, VPN, or internal ingress.
- Prometheus metrics enabled if observability stack exists.

Export and preserve before migration:

- listener configuration for `1883`, `18756`, and dashboard `18083`
- TLS certificates and listener TLS settings
- authentication backend and password/user database
- ACL/topic authorization rules
- retained messages
- persistent sessions and session expiry behavior
- plugins
- dashboard users and API tokens
- cluster cookie/node discovery settings
- any custom Erlang/EMQX config under `/etc/emqx` or equivalent

## Risks Or Constraints

- MQTT is production-critical and active. It should not be stopped or replaced without broker-level evidence, a test hub, and rollback.
- The backend uses QoS `0` while the upstream firmware spec says commands should use QoS `2`. A broker migration is a good time to expose this mismatch, but not a good time to change it without a separate protocol decision.
- The visible application code does not prove broker-side ACL enforcement. A Kubernetes migration must not weaken the trust boundary.
- Device clients may depend on hostname, port, certificate chain, public IP, reconnect timing, or retained/session state. These are currently open questions.
- EMQX version compatibility is a real decision. Running current Operator `2.3.x` cleanly points to EMQX `5.9+`, while the current broker is documented as `5.8.7`.
- Current API pods would each start MQTT and Kafka consumers. Scaling behavior must be tested before raising replica counts.
- The app currently uses PM2 and CodeDeploy conventions. Kubernetes should run one foreground process per container and rely on Kubernetes for restarts.
- No production Terraform/EKS files should be edited without explicit approval because infra file edits are approval-gated in this repo.

## Cost Optimization Opportunities

Kubernetes is not guaranteed to reduce cost. It may increase baseline cost because EKS control plane, node groups, load balancers, EBS volumes, and observability all add fixed cost.

The cost case is stronger if EKS consolidates multiple EC2 workloads and improves right-sizing. For EMQX specifically, the current `c5.xlarge` host appears underutilized by CPU and memory, so a smaller EC2 instance may be cheaper than EKS if cost is the only objective.

Use this decision rule:

- If the objective is lower cost only, right-size EMQX EC2 first after broker-level validation.
- If the objective is repeatable platform operations and HA, include EMQX in EKS phase 1 with a careful NLB and DNS cutover.

## Recommended Next Steps

1. Perform broker-level EMQX discovery before writing manifests:
   - `sudo emqx_ctl status`
   - `sudo emqx_ctl clients list`
   - listener, auth, ACL, plugin, retained message, and config export
   - dashboard read-only metrics if credentials are available
2. Inventory cron/background behavior on the current cron server and API hosts.
3. Add application readiness work to the backlog:
   - API and SSO `/healthz` and `/readyz`
   - graceful shutdown
   - stdout/stderr logging
   - container entrypoints without PM2
4. Build proof-of-concept images for API and SSO without changing production deployment.
5. Stand up a non-public EKS proof environment and validate connectivity to RDS, Redis, MongoDB Atlas, Kafka, S3, and EMQX.
6. Deploy EMQX in EKS behind a temporary hostname and validate with:
   - backend MQTT client
   - test hub/SIM
   - reconnect tests
   - retained/session tests
   - pod restart tests
7. Prepare a DNS cutover plan for `mqtt.sensorglobal.com`, including TTL reduction, old EC2 broker rollback, NLB health checks, and monitoring.

## Open Questions

- Do physical hubs pin a certificate, CA, public IP, or only the `mqtt.sensorglobal.com` hostname and port?
- Does each hub have unique EMQX credentials, or do hubs share credentials?
- Are ACLs currently enforced per controller serial number?
- Are retained messages or persistent sessions used for command delivery?
- What is the exact TLS state on ports `1883` and `18756`, and why was the previous TLS expiry tolerated by hubs?
- Is there a test hub with an active KORE SIM available for migration validation?
- Should EMQX be upgraded to `5.9`/`5.10` as part of the EKS migration, or should the first cutover preserve `5.8.7`?
- Should the first EKS cluster be in the existing prod VPC, a new VPC, or a future standalone account target?
