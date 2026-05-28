# EMQX Access And Dashboard Credentials

- Date: 2026-05-03
- Scope: production EMQX access through AWS SSM, live client checks, dashboard tunnel, and dashboard credential recovery
- Related systems: EMQX `Emqx-prod`, AWS SSM, AWS Secrets Manager `sensor-prod`, MQTT hub connectivity

## Objective

Document how to access the production EMQX broker safely, how to check whether hubs are connected, and how to recover dashboard access when no dashboard password is known.

## Sources Used

- Repository files:
  - `docs/infra/prod-services-access.md`
  - `docs/investigations/2026-04-10-inv-results.md`
  - `docs/investigations/2026-04-09-mqtt-device-communications.md`
  - `code/sensor-alarm-backend/src/config/app.ts`
  - `code/sensor-alarm-backend/src/services/mqtt/subscriber.ts`
- AWS commands or consoles:
  - `aws sts get-caller-identity`
  - `aws ec2 describe-instances`
  - `aws ssm start-session`
  - `aws secretsmanager list-secrets`
  - `aws secretsmanager get-secret-value` with key-name-only filtering
- External references: none

## Findings

Production EMQX is reachable through AWS SSM.

| Item | Value |
|---|---|
| AWS profile | `sensorsyn-mfa` |
| AWS region | `ap-southeast-2` |
| Account observed | `747293622182` |
| Instance name | `Emqx-prod` |
| Instance ID | `i-0b381a2bfdf65a329` |
| Private IP | `10.0.2.98` |
| Public IP | `13.211.102.112` |
| EMQX version checked | `5.8.7` |
| Dashboard port | `18083` |
| MQTT ports | `1883`, `18756` |

Live check on 2026-05-03 showed:

- `sudo emqx_ctl status` returned `Node 'emqx@127.0.0.1' 5.8.7 is started`.
- `sudo emqx_ctl clients list | wc -l` returned `18`.
- Hub clients appeared as serial-like client IDs such as `000905454949C0012406`.
- Backend/API clients appeared as `mqttjs_<random>`.

MQTT client credentials are in AWS Secrets Manager secret `sensor-prod` under:

- `MQTT_HOST`
- `MQTT_PORT`
- `MQTT_USERNAME`
- `MQTT_PASSWORD`

Those are not the same as EMQX Dashboard credentials. `sensor-prod` also contains `ADMIN_PASS`, but that key is used by the Sensor API bootstrap admin flow, not confirmed as an EMQX dashboard password.

## Read-Only EMQX Checks

Verify the AWS session first:

```bash
aws sts get-caller-identity \
  --profile sensorsyn-mfa \
  --region ap-southeast-2
```

Confirm the EMQX instance:

```bash
aws ec2 describe-instances \
  --profile sensorsyn-mfa \
  --region ap-southeast-2 \
  --instance-ids i-0b381a2bfdf65a329 \
  --query 'Reservations[0].Instances[0].{InstanceId:InstanceId,State:State.Name,PrivateIp:PrivateIpAddress,PublicIp:PublicIpAddress,Name:Tags[?Key==`Name`]|[0].Value}' \
  --output table
```

Check EMQX status:

```bash
aws ssm start-session \
  --profile sensorsyn-mfa \
  --region ap-southeast-2 \
  --target i-0b381a2bfdf65a329 \
  --document-name AWS-StartInteractiveCommand \
  --parameters '{"command":["sudo emqx_ctl status"]}'
```

Count connected clients:

```bash
aws ssm start-session \
  --profile sensorsyn-mfa \
  --region ap-southeast-2 \
  --target i-0b381a2bfdf65a329 \
  --document-name AWS-StartInteractiveCommand \
  --parameters '{"command":["sudo emqx_ctl clients list | wc -l"]}'
```

List a client sample:

```bash
aws ssm start-session \
  --profile sensorsyn-mfa \
  --region ap-southeast-2 \
  --target i-0b381a2bfdf65a329 \
  --document-name AWS-StartInteractiveCommand \
  --parameters '{"command":["sudo emqx_ctl clients list | head -n 30"]}'
```

Check whether one hub is connected:

```bash
HUB_SERIAL='<hub-serial>'

aws ssm start-session \
  --profile sensorsyn-mfa \
  --region ap-southeast-2 \
  --target i-0b381a2bfdf65a329 \
  --document-name AWS-StartInteractiveCommand \
  --parameters "{\"command\":[\"sudo emqx_ctl clients list | grep ${HUB_SERIAL}\"]}"
```

If a hub is connected, its client row should show:

- client ID matching or resembling the hub serial
- `connected=true`
- `subscriptions=1`
- a cellular/KORE-routed peer IP for field hubs

## MQTT Client Credentials

List matching secret keys without printing values:

```bash
aws secretsmanager get-secret-value \
  --profile sensorsyn-mfa \
  --region ap-southeast-2 \
  --secret-id sensor-prod \
  --query SecretString \
  --output text \
  | jq -r 'keys[]' \
  | sort \
  | rg -i 'emqx|mqtt|dashboard|user|pass|password'
```

Read MQTT fields when needed. Avoid pasting the values into tickets or chat:

```bash
aws secretsmanager get-secret-value \
  --profile sensorsyn-mfa \
  --region ap-southeast-2 \
  --secret-id sensor-prod \
  --query SecretString \
  --output text \
  | jq -r '.MQTT_HOST, .MQTT_PORT, .MQTT_USERNAME, .MQTT_PASSWORD'
```

## Dashboard Access

Open the dashboard tunnel:

```bash
aws ssm start-session \
  --profile sensorsyn-mfa \
  --region ap-southeast-2 \
  --target i-0b381a2bfdf65a329 \
  --document-name AWS-StartPortForwardingSession \
  --parameters '{"portNumber":["18083"],"localPortNumber":["18083"]}'
```

Then open:

```text
http://127.0.0.1:18083
```

Dashboard credentials are EMQX admin credentials, not MQTT client credentials.

## Dashboard Credential Recovery

Creating, deleting, or resetting an EMQX dashboard admin is a production credential mutation. It should only be done with explicit human approval. Prefer a temporary dashboard admin over resetting an unknown/default account.

The AWS CLI `--parameters` value must be JSON. This form works; do not use shell-quoted password/description arguments inside `command=...`, because AWS CLI parsing fails on the nested quotes.

Create a temporary dashboard admin:

```bash
aws ssm start-session \
  --profile sensorsyn-mfa \
  --region ap-southeast-2 \
  --target i-0b381a2bfdf65a329 \
  --document-name AWS-StartInteractiveCommand \
  --parameters '{"command":["sudo emqx_ctl admins add sensor-temp <strong-temp-password> temp-admin-2026-05-03"]}'
```

Known bad form that fails with `Error parsing parameter '--parameters'`:

```bash
aws ssm start-session \
  --profile sensorsyn-mfa \
  --region ap-southeast-2 \
  --target i-0b381a2bfdf65a329 \
  --document-name AWS-StartInteractiveCommand \
  --parameters command="sudo emqx_ctl admins add sensor-temp '<password>' 'Temporary admin for hub investigation 2026-05-03'"
```

Delete the temporary dashboard admin after use:

```bash
aws ssm start-session \
  --profile sensorsyn-mfa \
  --region ap-southeast-2 \
  --target i-0b381a2bfdf65a329 \
  --document-name AWS-StartInteractiveCommand \
  --parameters '{"command":["sudo emqx_ctl admins del sensor-temp"]}'
```

If a reset is explicitly preferred over a temporary user, the EMQX command shape is:

```bash
sudo emqx_ctl admins passwd <username> <new-password>
```

Run it through the same `AWS-StartInteractiveCommand` JSON wrapper.

## Risks Or Constraints

- The dashboard admin add/delete/reset commands mutate production broker credentials.
- MQTT client credentials are shared and powerful; do not print or paste them into documentation.
- Dashboard credentials and MQTT credentials are separate.
- Read-only `emqx_ctl clients list` is enough for most hub-connectivity investigations.
- Client IDs are not strong authentication proof by themselves because previous investigations found a shared broker credential and permissive ACL posture.

## Recommended Next Steps

1. If a temporary dashboard admin is created, delete it after the investigation.
2. Store the dashboard access procedure in an operational password manager rather than in repo docs.
3. For hub onboarding checks, prefer `emqx_ctl clients list | grep <hubSerial>` before using dashboard access.
4. Plan a later EMQX credential/ACL hardening investigation: per-device credentials or at least topic ACL restrictions.

## Open Questions

- What EMQX dashboard admin users currently exist?
- Is there an existing dashboard credential stored outside AWS Secrets Manager, such as a password manager?
- Should the team standardize on SSM/CLI read-only checks instead of dashboard access for day-to-day hub support?
