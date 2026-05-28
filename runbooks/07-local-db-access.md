# Runbook 07 — Local Database Access for Investigation

Provides read-only local access to all production data stores via AWS Systems Manager port forwarding. No VPN, no SSH keys required — only an active `sensorsyn-mfa` AWS CLI session.

## Prerequisites

```bash
# 1. Active MFA session (refresh if expired)
bash scripts/aws-mfa-login.sh default sensorsyn-mfa

# 2. AWS SSM plugin for local port forwarding
# macOS:  brew install session-manager-plugin
# Linux:  https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager-working-with-install-plugin.html
session-manager-plugin --version  # confirm installed
```

## How it works

SSM `AWS-StartPortForwardingSessionToRemoteHost` tunnels a local port through any EC2 instance in the VPC to any private endpoint that instance can reach. The EC2 instance does not need a public IP. All prod instances have the `SSM_For_EC2` profile and support this.

**Recommended jump host:** `i-019a920e221d70159` (`prod-api-server`, `10.0.5.178`) — private subnet, confirmed SSM-online on 2026-05-15, can reach prod VPC endpoints. If it is replaced by the Auto Scaling Group, use another online `prod-api-server` instance from SSM inventory.

---

## MySQL — `sensor-prod` (primary production database)

**Endpoint:** `sensor-prod.c4h1sisawrbr.ap-southeast-2.rds.amazonaws.com:3306`

### 1. Open the tunnel (keep this terminal open)

```bash
AWS_PROFILE=sensorsyn-mfa aws ssm start-session \
  --region ap-southeast-2 \
  --target i-019a920e221d70159 \
  --document-name AWS-StartPortForwardingSessionToRemoteHost \
  --parameters '{
    "host": ["sensor-prod.c4h1sisawrbr.ap-southeast-2.rds.amazonaws.com"],
    "portNumber": ["3306"],
    "localPortNumber": ["13306"]
  }'
```

### 2. Get credentials from Secrets Manager

```bash
AWS_PROFILE=sensorsyn-mfa aws secretsmanager get-secret-value \
  --region ap-southeast-2 \
  --secret-id sensor-prod \
  --query SecretString --output text \
  | python3 -m json.tool \
  | grep -E '"DB_MYSQL'
# Keys: DB_MYSQL_HOST, DB_MYSQL_PORT, DB_MYSQL_USERNAME, DB_MYSQL_PASSWPRD, DB_MYSQL_DBNAME
# NOTE: the password key is misspelled in the secret — it is DB_MYSQL_PASSWPRD, not DB_MYSQL_PASSWORD.
```

### 3. Connect (new terminal)

```bash
# mysql CLI
mysql -h 127.0.0.1 -P 13306 -u <DB_MYSQL_USERNAME> -p <DB_MYSQL_DBNAME>

# or TablePlus / DBeaver / DataGrip:
#   Host: 127.0.0.1   Port: 13306   DB: <DB_MYSQL_DBNAME>
```

**Key tables for investigation:** `tbl_admins`, `tbl_users`, `tbl_alarms`, `tbl_properties`, `tbl_jobs`, `tbl_notifications`, `tbl_sessions`

---

## PostgreSQL — `odoo-production`

**Endpoint:** `odoo-production.c4h1sisawrbr.ap-southeast-2.rds.amazonaws.com:5432`

### 1. Open the tunnel

```bash
AWS_PROFILE=sensorsyn-mfa aws ssm start-session \
  --region ap-southeast-2 \
  --target i-0b53bd5e7a9b9dec0 \
  --document-name AWS-StartPortForwardingSessionToRemoteHost \
  --parameters '{
    "host": ["odoo-production.c4h1sisawrbr.ap-southeast-2.rds.amazonaws.com"],
    "portNumber": ["5432"],
    "localPortNumber": ["15432"]
  }'
```

### 2. Get credentials from Secrets Manager

```bash
AWS_PROFILE=sensorsyn-mfa aws secretsmanager get-secret-value \
  --region ap-southeast-2 \
  --secret-id sensor-prod \
  --query SecretString --output text \
  | python3 -m json.tool \
  | grep -E '"ODOO|odoo'
# or check the Odoo EC2 instance env vars / Odoo config at /etc/odoo/odoo.conf
```

### 3. Connect

```bash
psql -h 127.0.0.1 -p 15432 -U <user> -d <db>
```

---

## MongoDB Atlas

**Cluster:** `sensor-prod.vr48f.mongodb.net` (external SaaS — no SSM tunnel needed)

### 1. Get connection string

```bash
AWS_PROFILE=sensorsyn-mfa aws secretsmanager get-secret-value \
  --region ap-southeast-2 \
  --secret-id sensor-prod \
  --query SecretString --output text \
  | python3 -m json.tool \
  | grep -E '"MONGO_DB_URL'
# MONGO_DB_URL contains the full mongodb+srv:// connection string with credentials
```

### 2. Allow your IP in Atlas Network Access

1. Log in to [cloud.mongodb.com](https://cloud.mongodb.com)
2. Go to **Network Access** → **Add IP Address** → add your current public IP
3. Or use **Allow Access from Anywhere** (`0.0.0.0/0`) for short investigation sessions

### 3. Connect

```bash
# mongosh (CLI)
mongosh "<MONGO_DB_URL>"

# MongoDB Compass (GUI):
#   Paste the MONGO_DB_URL connection string directly
```

**Key collections for investigation:** `activities`, `notifications`, `alarmLogs`, `deviceEvents`

---

## Redis — `smokeprodapiredis` (production cache)

**Endpoint:** `smokeprodapiredis.ngkvo9.ng.0001.apse2.cache.amazonaws.com:6379`

Redis is typically less useful for investigation (all keys have TTLs, mostly session tokens and short-lived state). But if you need it:

### 1. Open the tunnel

```bash
AWS_PROFILE=sensorsyn-mfa aws ssm start-session \
  --region ap-southeast-2 \
  --target i-0b53bd5e7a9b9dec0 \
  --document-name AWS-StartPortForwardingSessionToRemoteHost \
  --parameters '{
    "host": ["smokeprodapiredis.ngkvo9.ng.0001.apse2.cache.amazonaws.com"],
    "portNumber": ["6379"],
    "localPortNumber": ["16379"]
  }'
```

### 2. Connect

```bash
redis-cli -h 127.0.0.1 -p 16379
# No auth required (ElastiCache in prod VPC, no AUTH token configured)
```

---

## Non-production databases (same pattern)

| Environment | MySQL endpoint | Redis endpoint |
|---|---|---|
| QA | get via `aws rds describe-db-instances --query '...[?DBInstanceIdentifier==\`sensor-qa\`]...'` | `smokealarmqa.ngkvo9.ng.0001.apse2.cache.amazonaws.com:6379` |
| Sandbox | `sensor-sandbox` | `smokealarmsandbox.ngkvo9.ng.0001.apse2.cache.amazonaws.com:6379` |
| Dev | `dev-db-temp-mysql-8` | `smokealarmdevelopment.ngkvo9.ng.0001.apse2.cache.amazonaws.com:6379` |

Use the same SSM pattern but target an instance in the relevant environment VPC.

---

## Troubleshooting

| Issue | Resolution |
|---|---|
| `SessionManagerPlugin is not found` | Install the SSM session manager plugin (see Prerequisites) |
| `An error occurred (TargetNotConnected)` | The documented target may have been replaced by an Auto Scaling Group. Pick an online `prod-api-server` from `aws ssm describe-instance-information`; on 2026-05-15 `i-019a920e221d70159` and `i-0e1aaef0f8a051469` were online. |
| `An error occurred (ExpiredTokenException)` | Re-run `bash scripts/aws-mfa-login.sh default sensorsyn-mfa` |
| MySQL connection refused on 127.0.0.1:13306 | Confirm the SSM tunnel terminal is still running and shows `Waiting for connections...` |
| Atlas connection refused | Check your IP is in Atlas Network Access allowlist |
