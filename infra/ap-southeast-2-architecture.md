# ap-southeast-2 Architecture Summary

This page summarizes the current observed architecture in `ap-southeast-2` for account `747293622182`.

## Region role

- `ap-southeast-2` is an active primary region with separate VPCs for `prod`, `qa`, `dev`, `sandbox`, and `uat`.
- The visible application platform in this region is EC2 plus RDS, fronted by internet-facing ALBs.
- No ECS clusters were found in this region during the initial baseline.

## Environment topology

| Environment | VPC name | VPC ID | Notes |
| --- | --- | --- | --- |
| prod | `Smoke-VPC` | `vpc-03ec09702866ca7b1` | busiest footprint, dedicated RDS private subnets, multiple ALBs |
| qa | `Smoke-qa-VPC` | `vpc-0682d7403418ab9db` | mirrors production patterns with QA-specific ALBs, APIs, and databases |
| dev | `Smoke-development-VPC` | `vpc-03e1525483229726e` | partial active footprint, several stopped hosts |
| sandbox | `Smoke-Sandbox-VPC` | `vpc-0c4eb38fb674b072b` | active API, SSO, Odoo, Kafka, and EMQX-related hosts |
| uat | `smoke-uat-vpc` | `vpc-07917d0b4878f2c15` | separate CIDR range, smaller active footprint |

## Network pattern

- Each non-default environment VPC has its own internet gateway and one public NAT gateway.
- `prod`, `qa`, `dev`, and `sandbox` use a common CIDR and subnet structure:
  - public subnets in `10.0.1.0/24`, `10.0.2.0/24`, `10.0.3.0/24`
  - private subnets in `10.0.4.0/24`, `10.0.5.0/24`, `10.0.6.0/24`
- `uat` uses `172.27.64.0/18` with separate private and public subnet ranges.
- `prod` also includes dedicated RDS private subnets.

## Workload pattern

- Application ingress appears to be ALB-based, with separate internet-facing ALBs for API and SSO stacks in most environments.
- Production also has dedicated ALBs for keychain and WordPress.
- EC2 hosts include API servers, SSO, OpenVPN, Kafka, EMQX, Odoo, WordPress, and support systems such as Zabbix and Power BI Gateway.
- RDS is environment-scoped and placed in private subnet groups per VPC.
- Multi-AZ RDS was observed for `sensor-prod`, `sensor-sandbox`, and `smokealaramuat`.

## Current cautions

- Several non-production environments remain substantially provisioned, which may have both cost and governance implications.
- Some EC2 instances have public IPs even though the named public subnets report `MapPublicIpOnLaunch = false`, so public exposure should be reviewed carefully.
- The default VPC still exists and should be checked for legacy or orphaned resources in a later pass.

Related investigation: [2026-04-09-ap-southeast-2-architecture-baseline.md](/home/yevgen/dev/sensorsyn/docs/investigations/2026-04-09-ap-southeast-2-architecture-baseline.md)
