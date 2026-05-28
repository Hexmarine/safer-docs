# Production Services — Access Reference

**Account:** `747293622182` (ap-southeast-2)  
**Last verified:** 2026-04-12  
**All EC2 instances** have the `SSM_For_EC2` instance profile — shell access via SSM to every instance, no SSH keys or VPN required.

---

## Quick access — SSM shell to any instance

```bash
AWS_PROFILE=sensorsyn-mfa aws ssm start-session --region ap-southeast-2 --target <instance-id>
```

## Quick access — databases (local tunnels)

```bash
./scripts/db-tunnel-start.sh        # opens all three tunnels in background
./scripts/db-tunnel-stop.sh status  # check what's running
./scripts/db-tunnel-stop.sh         # close all

# Then connect:
mysql -h 127.0.0.1 -P 13306 -u admin -p smokealarmprod
psql "host=127.0.0.1 port=15432 dbname=<db> user=<user>"
redis-cli -h 127.0.0.1 -p 16379
```

---

## Production Application Inventory

### 1. Angular web portals — `CloudFront → S3`

All served from S3 bucket `smokealarmprod-admin.s3.ap-southeast-2.amazonaws.com` via a single CloudFront distribution.

| URL | Purpose |
|---|---|
| https://admin.sensorglobal.com | Agency / admin portal |
| https://agent.sensorglobal.com | Real estate agent portal |
| https://trader.sensorglobal.com | Contractor / tradesperson portal |
| https://owner.sensorglobal.com | Property owner portal |
| https://activation.sensorglobal.com | Device activation flow |
| https://reschedule.sensorglobal.com | Job rescheduling flow |

**Access:** Browser — publicly accessible, no special setup needed.

---

### 2. Node.js API — `EC2 × 5 + ALB`

| Item | Value |
|---|---|
| Public URL | https://api.sensorglobal.com/api/v1/ |
| ALB | `SmokeAPI-LB-898969882.ap-southeast-2.elb.amazonaws.com` |
| Instances (all private, t3.medium) | see table below |

| Instance ID | Private IP | Name |
|---|---|---|
| `i-0277626c3f4828de7` | 10.0.5.213 | prod-api-server |
| `i-035d91e3857aaea0e` | 10.0.5.14 | prod-api-server |
| `i-09acc276e638ba099` | 10.0.4.206 | prod-api-server |
| `i-0b53bd5e7a9b9dec0` | 10.0.6.25 | prod-api-server ← preferred SSM jump host |
| `i-0bcc2dcab7c6dfedf` | 10.0.4.248 | prod-api-server |

> ⚠️ **Instance IDs churn.** The API runs behind a CodeDeploy/Auto Scaling Group, so the IDs above are point-in-time (table last verified 2026-04-12) and will not match after a deploy or scale event. Get the current fleet before relying on a specific ID:
>
> ```bash
> AWS_PROFILE=sensorsyn-mfa aws ssm describe-instance-information \
>   --region ap-southeast-2 \
>   --query "InstanceInformationList[?PingStatus=='Online'].InstanceId" --output text
> ```
>
> Cross-check names with `aws ec2 describe-instances --instance-ids <ids> --query "Reservations[].Instances[].[InstanceId,Tags[?Key=='Name']|[0].Value]" --output text`. The same applies to the SSM jump host used by the DB tunnels — see `runbooks/07-local-db-access.md` (override with `JUMP_HOST=<id> ./scripts/db-tunnel-start.sh` if the default target is `TargetNotConnected`).

**API access:** Browser/curl to `https://api.sensorglobal.com/api/v1/`. Shell via SSM to any instance.

**App logs:**
```bash
AWS_PROFILE=sensorsyn-mfa aws ssm start-session --region ap-southeast-2 --target i-0b53bd5e7a9b9dec0
# then on the instance: sudo journalctl -u <service> -f  or  sudo pm2 logs
```

---

### 3. SSO / Authentication — `EC2 + ALB`

| Item | Value |
|---|---|
| Public URL | https://auth.sensorglobal.com |
| ALB | `sso-api-lb-1719844694.ap-southeast-2.elb.amazonaws.com` |
| Instance | Current as of 2026-05-08: `i-01606ffe2138c0c8e` (10.0.5.248, t3.small, private), launched by ASG `CodeDeploy_sso-api-dg_d-QKLSZP50D` |

**Access:** Browser. Shell via `aws ssm start-session --target i-01606ffe2138c0c8e`.

---

### 4. Keychain — `EC2 + ALB`

| Item | Value |
|---|---|
| Public URL | https://keychain.sensorglobal.com |
| ALB | `keychain-production-ALB-496662693.ap-southeast-2.elb.amazonaws.com` |
| Instance | `i-07b498016eab37ae1` (10.0.4.16, t3.small, private) |

> ⚠️ **Shared across all environments** — `keychain-development.`, `keychain-qa.`, `keychain-sandbox.` all point to the same ALB and instance. A change here affects all environments simultaneously.

**Access:** Browser. Shell via `aws ssm start-session --target i-07b498016eab37ae1`.

---

### 5. Odoo CRM — `EC2, direct public IP`

| Item | Value |
|---|---|
| URLs | https://sensorglobal.com (root A record), https://crm.sensorglobal.com |
| Public IP | `54.253.158.197` |
| Instance | `i-0c5073e95aba77614` (10.0.2.161, t3.large) |
| Database | `odoo-production` PostgreSQL (via DB tunnel on 15432) |

**Access:** Browser directly. Admin login at `sensorglobal.com/web`. Shell via SSM.

```bash
AWS_PROFILE=sensorsyn-mfa aws ssm start-session --region ap-southeast-2 --target i-0c5073e95aba77614
```

---

### 6. WordPress (marketing + web.sensorglobal.com) — `EC2 + ALB`

| Item | Value |
|---|---|
| URL | https://web.sensorglobal.com |
| ALB | `wordpress-prod-alb-1756963168.ap-southeast-2.elb.amazonaws.com` |
| Instance | `i-0bd9a5e391e812fcc` (10.0.4.29, t3.medium, private) |

**Access:** Browser. Shell via `aws ssm start-session --target i-0bd9a5e391e812fcc`.

---

### 7. EMQX MQTT Broker — `EC2, public IP`

| Item | Value |
|---|---|
| Hostname | `mqtt.sensorglobal.com` → `13.211.102.112` |
| Instance | `i-0b381a2bfdf65a329` (10.0.2.98, c5.xlarge) |
| Hub MQTT | Port **1883** (within VPC + KORE SIM CIDRs) |
| External MQTT | Port **18756** (KORE/ONO SIM CIDRs only) |
| Dashboard | Port **18083** (firewalled externally — use SSM tunnel) |
| Version | EMQX 5.8.7 |

**Shell access:**
```bash
AWS_PROFILE=sensorsyn-mfa aws ssm start-session --region ap-southeast-2 --target i-0b381a2bfdf65a329
# on instance: sudo emqx_ctl clients list | wc -l
# on instance: sudo emqx_ctl status
```

**Dashboard (web UI) — SSM tunnel required:**
```bash
AWS_PROFILE=sensorsyn-mfa aws ssm start-session \
  --target i-0b381a2bfdf65a329 \
  --document-name AWS-StartPortForwardingSession \
  --parameters '{"portNumber":["18083"],"localPortNumber":["18083"]}'
# then open http://localhost:18083 in browser
```

---

### 8. Kafka Broker — `EC2, public IP`

| Item | Value |
|---|---|
| Hostname | `kafka.sensorglobal.com` → `13.237.253.114` |
| Instance | `i-09d176fd791fe7c64` (10.0.1.199, t3.medium) |
| Port | **9092** PLAINTEXT (no auth, no TLS) |
| Version | Kafka 3.0.0 (ZooKeeper mode) |

> ⚠️ `log.dirs=/tmp/kafka-logs` — Kafka data is in `/tmp` and will be lost on instance reboot.

**Shell access:**
```bash
AWS_PROFILE=sensorsyn-mfa aws ssm start-session --region ap-southeast-2 --target i-09d176fd791fe7c64
# on instance: /opt/kafka/bin/kafka-topics.sh --bootstrap-server localhost:9092 --list
```

**Local Kafka access (SSM tunnel):**
```bash
AWS_PROFILE=sensorsyn-mfa aws ssm start-session \
  --target i-09d176fd791fe7c64 \
  --document-name AWS-StartPortForwardingSession \
  --parameters '{"portNumber":["9092"],"localPortNumber":["19092"]}'
# then: kcat -b localhost:19092 -L
```

---

### 9. Cron Server — `EC2`

| Item | Value |
|---|---|
| Instance | `i-0a1cd47cfadedf035` (10.0.2.167, t2.micro) |
| Public IP | `54.153.177.72` |
| Purpose | Scheduled background jobs for the API |

**Access:**
```bash
AWS_PROFILE=sensorsyn-mfa aws ssm start-session --region ap-southeast-2 --target i-0a1cd47cfadedf035
# on instance: crontab -l  or  sudo pm2 list
```

---

### 10. MongoDB Atlas — `External SaaS`

| Item | Value |
|---|---|
| Cluster | `sensor-prod.vr48f.mongodb.net` |
| Credentials | `MONGO_DB_URL` in Secrets Manager secret `sensor-prod` |

**Access:**
```bash
# Retrieve URI
AWS_PROFILE=sensorsyn-mfa aws secretsmanager get-secret-value \
  --region ap-southeast-2 --secret-id sensor-prod \
  --query SecretString --output text | python3 -c "import sys,json; d=json.load(sys.stdin); print(d.get('MONGO_DB_URL',''))"

# Connect
mongosh "<uri from above>"
```

---

## Databases (via DB tunnel scripts)

| Database | Engine | SSM tunnel port | Connection |
|---|---|---|---|
| `smokealarmprod` | MySQL 8.0 (RDS, Multi-AZ) | **13306** | `mysql -h 127.0.0.1 -P 13306 -u admin -p smokealarmprod` |
| `odoo` | PostgreSQL 14.17 (RDS, single-AZ) | **15432** | `psql "host=127.0.0.1 port=15432 dbname=odoo user=<user>"` |
| Redis cache | ElastiCache (2-node cluster) | **16379** | `redis-cli -h 127.0.0.1 -p 16379` |

> MySQL credentials: user `admin`, password from `DB_MYSQL_PASSWPRD` key (typo!) in secret `sensor-prod`.

---

## OpenVPN

| Item | Value |
|---|---|
| Instance | `i-00abd001f35108630` (t2.micro) |
| Public IP | `13.211.118.57` |

Legacy access method — SSM is preferred for all investigations and avoids the need to manage VPN credentials.

---

## Non-prod environments (same account)

| Environment | API URL | Angular portal |
|---|---|---|
| QA | api-qa.sensorglobal.com | admin-qa.sensorglobal.com |
| Sandbox | api-sandbox.sensorglobal.com | admin-sandbox.sensorglobal.com |
| Development | api-dev.sensorglobal.com | admin-development.sensorglobal.com |
| UAT | (via ALB) | admin-uat.sensorglobal.com |
