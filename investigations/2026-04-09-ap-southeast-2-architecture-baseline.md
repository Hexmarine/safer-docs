# ap-southeast-2 Architecture Baseline

- Date: 2026-04-09
- Scope: current-state network, compute, and database layout in `ap-southeast-2`
- Related systems: VPC, subnets, NAT/IGW, ALB, EC2, RDS for account `747293622182`

## Objective

Establish a durable architecture baseline for the active `ap-southeast-2` footprint so later security, reliability, and cost investigations can build on a clear current-state map.

## Sources Used

- Repository files: `AGENTS.md`, `docs/README.md`, `docs/investigations/2026-04-09-aws-account-baseline.md`
- AWS commands or consoles:
  - `aws ec2 describe-vpcs --region ap-southeast-2`
  - `aws ec2 describe-subnets --region ap-southeast-2`
  - `aws ec2 describe-route-tables --region ap-southeast-2`
  - `aws ec2 describe-internet-gateways --region ap-southeast-2`
  - `aws ec2 describe-nat-gateways --region ap-southeast-2`
  - `aws elbv2 describe-load-balancers --region ap-southeast-2`
  - `aws ec2 describe-instances --region ap-southeast-2`
  - `aws rds describe-db-instances --region ap-southeast-2`
  - `aws ec2 describe-security-groups --region ap-southeast-2`
- External references: none

## Findings

- `ap-southeast-2` is an active multi-environment region with separate VPCs for `prod`, `qa`, `dev`, `sandbox`, and `uat`.
- The environment mapping is supported by both VPC naming and workload placement, not by names alone.
- No ECS clusters were found earlier in this region, so the visible workload plane is primarily EC2 plus RDS, fronted by internet-facing ALBs.
- The default VPC exists but no workload from this pass appears to depend on it directly.

### VPC and environment mapping

- `vpc-03ec09702866ca7b1` `Smoke-VPC`: production
- `vpc-0682d7403418ab9db` `Smoke-qa-VPC`: QA
- `vpc-03e1525483229726e` `Smoke-development-VPC`: development
- `vpc-0c4eb38fb674b072b` `Smoke-Sandbox-VPC`: sandbox
- `vpc-07917d0b4878f2c15` `smoke-uat-vpc`: UAT
- `vpc-ae16eec8` `default`: legacy/default footprint, not part of the main application topology based on current evidence

### Network shape

- Each non-default environment VPC has its own internet gateway.
- Each non-default environment VPC also has one public NAT gateway, suggesting a consistent pattern of private workloads with managed outbound internet access.
- `prod`, `qa`, `dev`, and `sandbox` follow the same broad subnet shape:
  - public subnets using `10.0.1.0/24`, `10.0.2.0/24`, and `10.0.3.0/24`
  - private subnets using `10.0.4.0/24`, `10.0.5.0/24`, and `10.0.6.0/24`
- `prod` additionally contains three dedicated RDS private subnets:
  - `subnet-0949bfb4548081f25` `10.0.0.128/25`
  - `subnet-0ec989f2f61cb5f25` `10.0.7.0/25`
  - `subnet-0594cfdc32c9dbe6c` `10.0.7.128/25`
- `uat` uses a different CIDR structure:
  - private subnets at `172.27.64.0/20`, `172.27.80.0/20`, and `172.27.96.0/20`
  - public subnets at `172.27.112.0/21`, `172.27.120.0/22`, and `172.27.124.0/22`
- Route table counts are consistent with public/private segmentation in each non-default VPC.

### Load-balancing and ingress

- All discovered load balancers in `ap-southeast-2` are application load balancers.
- All discovered ALBs are `internet-facing`.
- Observed ALB naming indicates separate entry points for:
  - core API workloads in `prod`, `qa`, `dev`, `sandbox`, and `uat`
  - SSO workloads in `prod`, `qa`, `dev`, `sandbox`, and `uat`
  - WordPress in `prod` and `qa`
  - keychain in `prod`
- This suggests environment isolation is implemented per VPC while external exposure is provided at the ALB layer.

### EC2 workload placement

- Production VPC contains the busiest footprint, including:
  - multiple running `prod-api-server` instances in private subnets
  - `prod-sso-api` in a private subnet
  - running `prod-kafka`, `Emqx-prod`, `openvpn-prod`, `Odoo-production`, `Smoke-prod-Keychain-app`, and `wordpress-prod`
  - `Power-BI-Gateway` and `wordpress-sensor-insure`
- QA VPC contains:
  - multiple running `smoke-qa-api` instances in private subnets
  - running `sso-qa-api`, `openvpn-qa`, `wordpress-qa`, `odoo-qa-*`
  - stopped and running Kafka and EMQX related instances
- Development VPC contains:
  - running `dev-sso-api` and `zabbix`
  - stopped `dev-api-server`, Kafka, SonarQube, Odoo, and EMQX-related instances
- Sandbox VPC contains:
  - running `sandbox-api-server`, `sandbox-sso-api`, `openvpn-sandbox`, `Emqx-sandbox`, Kafka-related nodes, and `Odoo-sandbox`
- UAT VPC contains:
  - running `uat-api-server`
  - stopped `openvpn-uat`, `Emqx-uat`, `odoo-uat-server`, and `uat-kafka`
- Many application servers in `prod`, `qa`, `dev`, `sandbox`, and `uat` are private-only instances, while several infrastructure-style nodes still have public IPs.
- Publicly addressed instances were observed for OpenVPN, EMQX, some Odoo hosts, Kafka-related nodes, and a cron server. This likely reflects deliberate manual association rather than subnet auto-assignment because all named public subnets reported `MapPublicIpOnLaunch = false`.

### RDS placement

- Eight RDS instances were discovered in `ap-southeast-2`.
- Production VPC:
  - `sensor-prod` MySQL `db.t3.medium`, `MultiAZ = true`
  - `odoo-production` PostgreSQL `db.t3.medium`, `MultiAZ = false`
- QA VPC:
  - `sensor-qa` MySQL `db.t3.medium`, `MultiAZ = false`
  - `odoo-qa` PostgreSQL `db.t3.medium`, `MultiAZ = false`
- Development VPC:
  - `dev-db-temp-mysql-8` MySQL `db.t3.micro`, `MultiAZ = false`
  - `odoo-dev` PostgreSQL `db.t3.medium`, `MultiAZ = false`
- Sandbox VPC:
  - `sensor-sandbox` MySQL `db.t3.medium`, `MultiAZ = true`
- UAT VPC:
  - `smokealaramuat` MySQL `db.t3.medium`, `MultiAZ = true`
- RDS subnet groups align cleanly to environment-specific private subnet groups, which supports the per-VPC environment model.

### Security-group observations

- Security groups also appear segmented by environment and workload role.
- Naming suggests separate groups for ALBs, API servers, SSO, RDS, OpenVPN, Kafka, EMQX, WordPress, ElastiCache, Lambda, and client VPN.
- The volume of named security groups suggests a manually evolved environment with service-by-service rule sets rather than a single centralized network pattern.

## Risks Or Constraints

- Environment mapping is strong but still partly inferred from naming conventions and placement; tags were not yet validated beyond `Name`.
- Internet-facing ALBs exist in every major environment, so later security work should confirm whether non-production exposure is intentional.
- Several non-prod and infrastructure-style EC2 instances still have public IPs, which may broaden attack surface and increase operational variance.
- Production data-plane coverage is incomplete from this pass because route details, target groups, DNS, and certificate bindings were not yet inspected.
- The default VPC still contains security groups and should be checked later for orphaned or legacy resources.

## Cost Optimization Opportunities

- Stopped instances exist across `dev`, `qa`, and `uat`, especially around Kafka, Odoo, EMQX, OpenVPN, and API hosts. These should be reviewed for snapshot-and-delete or schedule-based shutdown opportunities.
- Environment duplication is substantial. If some `qa`, `sandbox`, and `uat` services are lightly used, rightsizing or consolidation may be possible.
- Public NAT gateways are deployed per environment VPC. This is operationally common, but their actual traffic volume should be checked before assuming the cost is justified.
- Several always-on EC2 workloads in non-prod environments may be candidates for instance schedule controls if they do not require continuous uptime.

## Recommended Next Steps

- Document ALB listeners, target groups, and attached instances to make ingress paths explicit.
- Inspect route tables in detail to confirm which subnets are truly public and how NAT is wired per environment.
- Investigate Route 53, ACM, and CloudFront to connect public DNS names and certificates to the ALB layer.
- Validate security groups around public-facing instances and ALBs, prioritizing `prod`, then `qa`, `sandbox`, and `uat`.
- Investigate the default VPC for leftover legacy compute, databases, or public exposure.

## Open Questions

- Which ALBs and EC2 groups serve the primary customer-facing production application versus supporting systems such as WordPress, SSO, Odoo, and MQTT/EMQX?
- Are `qa`, `sandbox`, and `uat` all still actively used, or do some represent low-value duplicated environments?
- Why do several named public subnets disable auto public IP assignment while some instances in those subnets still have public IPs?
- What resources, if any, remain active in the default VPC beyond security groups?
