# Applied Changes

Record only approved changes that were actually applied.

### 2026-07-02 — sensor-alarm-backend log-noise fix deployed (95% of prod log volume)
- **Approval reference:** cost session; Logs Insights showed `smoke-api-prod-pm2-out-log` at ~5.3GB/day = 54% KORE SIM-webhook console dumps (incl. imei/imsi + signature material) + 43% Sequelize SQL logging (`logging: true` in BootStrap.ts) + <2% the pino access log (exonerated — the mid-June step-up was the Haven SIM-swap webhook storm, not structured logging). User committed and deployed.
- **Change (4 files, regression-guard verdict SAFE, tsc clean):** Sequelize logging env-gated (`LOG_SQL=1`), all 3 `mongoose.set("debug")` sites gated (`MONGOOSE_DEBUG=1`), KORE signature-value logs and SIM payload dump removed (also closes the PII/secret-material leak), checkMongoConnection poll spam removed.
- **Verification (post-deploy d-MXFXL1QBJ, Succeeded 11:20Z):** `Executing (default)` lines 3,400/10min → 63/10min (old-instance tail), expect ~0 steady-state; ~5.4→~0.3GB/day ingestion expected within 24h (~$85-90/mo). Same deploy was the golden-AMI pruner's first live run: new AMI baked, LT v744, 3 AMIs+snapshots pruned (517→515) — retention automation confirmed working end-to-end.
- **Undo:** set `LOG_SQL=1`/`MONGOOSE_DEBUG=1` in env (no redeploy of code needed beyond restart) or revert the commit.
- **Caveat:** the removed signature logs were the only KORE-401 diagnostic; if a webhook signature dispute recurs, re-add a redacted env-gated version.

### 2026-07-02 — GuardDuty HIGH findings wired to email (EventBridge → SNS)
- **Approval reference:** cost-cleanup session, user "lets wire guardduty" (follow-up to keeping GuardDuty as the sole security service — previously its findings landed silently in the console).
- **Applied by:** Claude, ap-southeast-2: SNS topic `guardduty-high-findings` (topic policy allows events.amazonaws.com publish) + email subscription `eugene.peresada@saferhomesau.com.au` (confirmed; an earlier `peresada@gmail.com` subscription was never confirmed and auto-expires in 3 days) + EventBridge rule `guardduty-high-findings` matching `source=aws.guardduty, detail.severity >= 7` with an InputTransformer producing a readable plaintext email incl. a direct console deep-link to the finding.
- **GOTCHA:** EventBridge InputTemplate in quoted-string form must contain literal `\n` two-char sequences — real newlines inside the JSON string fail validation.
- **Verification:** END-TO-END CONFIRMED — fired `create-sample-findings` (`Backdoor:EC2/C&CActivity.B!DNS`, sev 8, `[SAMPLE]`-marked), user received the formatted email at the saferhomesau address within minutes; sample finding archived afterwards to keep the console clean.
- **Undo:** delete the EventBridge rule + SNS topic.
- **Cost:** negligible (SNS email is free tier scale; EventBridge rules are free for AWS-service events).

### 2026-07-02 — sensor-prod-old1 deleted (post-upgrade soak ended early, all healthy)
- **Approval reference:** cost-cleanup session; user "and old rds can decommission, looks like all behaves ok" — ending the planned ~1-week soak from the 2026-06-30 MySQL 8.4 upgrade 5 days early on observed health. **User ran both mutation commands via `!`** (harness blocks Claude from RDS deletion regardless of approval): deletion-protection off → `delete-db-instance --skip-final-snapshot`.
- **Pre-checks (Claude, read-only):** confirmed target identity (8.0.44 = the old blue, DeletionProtection was true) and that safety snapshot `pre-mysql84-upgrade-20260630-1116` is `available` (created 11:16Z, ~47 min before the 12:03Z switchover — near-identical data since blue stopped replicating at cutover). No final snapshot taken for that reason.
- **Verification:** `sensor-prod-old1` status `deleting`; live `sensor-prod` (8.4.8) and `odoo-production` untouched and available.
- **Undo:** restore from the pre-upgrade snapshot (loses nothing that matters — blue received no writes after cutover).
- **Saving:** ~$120/mo (Multi-AZ db.t3.medium + storage). Follow-up: the pre-upgrade snapshot itself (~$0.50/mo) can be deleted after another soak week if desired.

### 2026-07-02 — Compliance stack disabled: SecurityHub + Inspector + Config off, GuardDuty retained
- **Approval reference:** cost-cleanup session; user explicitly chose "Drop all 3, keep GuardDuty" via AskUserQuestion, then "go security stack". Rationale: solo operator, nobody triages hygiene findings; GuardDuty stays (active-threat detection; its last-90d HIGH findings were reviewed live and are real signal — see session notes).
- **Applied by:** Claude, ap-southeast-2 (all $149/mo of the 3 services' cost was in this one region): `securityhub disable-security-hub`; `inspector2 disable` (EC2/ECR/Lambda/LambdaCode → DISABLING); Config `stop-configuration-recorder` (recorder `default`); bulk-deleted the 484 standalone Config rules — 126 user-created deleted, the other 358 are `securityhub-*` service-linked rules that AccessDenied direct deletion and are auto-removed by AWS asynchronously after the SecurityHub disable (re-check in ~24h that they drained).
- **Undo:** all reversible — re-enable each service; historical findings/config-item history are lost. Config delivery channel + recorder definition left in place (stopped), so `start-configuration-recorder` restores recording if ever wanted.
- **Saving:** ~$150/mo (Inspector $53 + Config $51 + SecurityHub $46, June).

### 2026-07-02 — QuickSight Enterprise subscription cancelled (empty — zero dashboards)
- **Approval reference:** cost-cleanup session, user "go quicksight". Subscription was ENTERPRISE edition with 0 published dashboards, 13 never-published draft analyses, 6 users (one a `security-hub-finding` service account; notification email still pointed at the original dev agency appinventiv) — set up and never operationalized.
- **Applied by:** Claude: `update-account-settings --no-termination-protection-enabled` then `delete-account-subscription` (status 202 → `UNSUBSCRIBE_IN_PROGRESS`).
- **Undo:** none — cancellation permanently discards the draft analyses/users. Accepted explicitly.
- **Saving:** ~$102/mo.

### 2026-07-02 — Power-BI-Gateway terminated (stopped since Dec 2024)
- **Approval reference:** cost-cleanup session, user "go for powerbi gateway". Instance `i-0908eff051972cd8c` (t3.large) had been stopped 18 months; its 50GB gp3 volume was the only remaining cost. Power BI reporting has been de facto dead that whole time (the `power.bi` MySQL user remains in sensor-prod, unused).
- **Applied by:** Claude, AWS CLI: safety snapshot `snap-010c70d9b39dd01fa` (vol-0cfdcc56eebc91ac6, waited completed) → termination protection off → terminated → deleted its CPU alarm.
- **Undo:** restore from snapshot.
- **Saving:** ~$5/mo.

### 2026-07-02 — Prod Redis right-sized: 2× cache.t3.medium → cache.t3.small (smokeprodapiredis)
- **Approval reference:** cost-cleanup session; 14-day peak usage 39MB of 3,090MB (1.3%), engine CPU peak 0.45%, 17 connections. User selected "cache.t3.small" explicitly via AskUserQuestion, then "lets go one by one".
- **Applied by:** Claude, `elasticache modify-replication-group --cache-node-type cache.t3.small --apply-immediately`. Rolling resize with auto-failover (replica first), ~25 min total, completed 07:39Z.
- **Verification:** both nodes `available` on t3.small; CurrConnections held steady (~10) through the entire resize — no client drop; both prod API targets healthy behind SmokeAPI-LB afterwards. (Public health-check probe returns 403 from the workstation — that's WAF, not the API.)
- **Undo:** same command back to `cache.t3.medium`.
- **Saving:** ~$220/mo (~75% of the ~$294 ElastiCache line).

### 2026-07-02 — Keychain app decommissioned (empty instance + ALB serving nothing)
- **Approval reference:** cost-cleanup session; user chose "Investigate first" then acked "its a go for keychain" after the SSM investigation showed instance `i-07b498016eab37ae1` (Smoke-prod-Keychain-app, t3.small, running since Mar 2024) ran NO application: no app processes, only sshd listening, empty /var/www/html, no pm2/nginx. Its ALB had 155 requests in 14 days (scanner noise).
- **Applied by:** Claude, AWS CLI. Order: root-volume safety snapshot `snap-06fb6438993a0fde3` (vol-011a40eebb7e65b65, waited completed) → deleted 4 Route53 CNAMEs (`keychain{,-development,-qa,-sandbox}.sensorglobal.com`, all pointing at the one prod ALB — removed to avoid dangling records) → terminated instance (needed `--no-disable-api-termination` first) → deleted ALB `keychain-production-ALB` (needed deletion-protection off first) → deleted target group `keychain-prod-Target-group` → deleted its 2 CloudWatch alarms.
- **Undo:** instance restorable from the snapshot; DNS/ALB would need manual recreation (nothing was being served, so nothing to restore in practice).
- **Saving:** ~$45/mo (ALB ~$23 + t3.small ~$19 + EBS).

### 2026-07-02 — CWAgent disk-metric junk filter rolled out to 7 prod hosts (SSM param AmazonCloudWatch-linux)
- **Approval reference:** same CloudWatch-economy session; user acked "its a go for aws cloudwatch" for the named shared parameter after the harness classifier (correctly) blocked the first attempt on a vague "yes pls". Root cause verified live pre-change: shared CWAgent config used `disk.resources: ["*"]` with no filesystem filter → 494 of 665 CWAgent custom metrics (76%) were junk series for snap squashfs loops, Docker overlay layers, tmpfs/devtmpfs across 7 hosts (prod-kafka, Smoke-prod-Keychain-app, Odoo-production, Emqx-prod, wordpress-prod, wordpress-sensor-insure, smoke-prod-api-cron-server).
- **Change (2 param versions):** v7→v8 added `"ignore_file_system_types": ["tmpfs","devtmpfs","sysfs","squashfs","overlay"]` to the `disk` block (only change). v8→v9 added `["InstanceId","path","device","fstype"]` to `aggregation_dimensions` — needed because the restarted agents publish full-dimension series (+ImageId/InstanceType) and the 6 existing per-host disk alarms are keyed to the old 4-dim series, which would have gone INSUFFICIENT_DATA; the rollup re-emits exactly those series (and survives future AMI changes, unlike re-pointing alarms at ImageId-tagged series).
- **Rollout:** canary `wordpress-sensor-insure` first, then the other 6 via SSM RunShellScript `amazon-cloudwatch-agent-ctl -a fetch-config -m ec2 -s -c ssm:AmazonCloudWatch-linux`. GOTCHA: first canary restart failed validation — the shared config's `collectd` section requires `/usr/share/collectd/types.db`, absent on these hosts (agents had been running year-old configs and never re-validated); fixed with an empty stub file on all 7 (AWS-documented workaround). All 7 validated + restarted cleanly.
- **Verification:** canary junk series (squashfs/tmpfs combos) have zero post-restart datapoints while real `/` series + InstanceId rollups flow; after v9, the exact alarm-keyed 4-dim series confirmed publishing on kafka/emqx/cron hosts with only a ~3-min restart gap (alarms never flipped, all 6 still OK). Emqx broker verified healthy post-restart (active; 1883/8883 listening) — agent is a sidecar, app processes untouched.
- **Undo:** restore param v7 from scratchpad copy (`cw-linux-current.json`) via put-parameter, re-run the same fetch-config on the 7 hosts.
- **Saving:** ~494 junk custom-metric series stop reporting (minus ~30 new rollup series added for alarm compatibility); expect most of the ~$48/mo MetricMonitorUsage line to fall away — confirm in Cost Explorer after a few days.
- **Note:** the ~256 `host`-dimensioned CWAgent series (ip-10-0-4-x) come from the prod-api ASG nodes, which do NOT use this SSM param (baked AMI config) — untouched, separate lever if wanted.

### 2026-07-01 — Deleted 77 orphaned CloudWatch alarms (INSUFFICIENT_DATA cleanup)
- **Approval reference:** same "CloudWatch usage/economy" session. Of 181 alarms, 112 (later 109, some transitioned) were stuck `INSUFFICIENT_DATA`. Sampled 11 at random, all 11 pointed at confirmed-dead resources; user said "lets do number 2" (proceed with verify+delete). Ran a full classification of all 109 against live inventory (EC2, ASG, Lambda, RDS, ALB, target group) requiring *every* resource an alarm references to be confirmed-gone before calling it dead (conservative — avoids deleting an alarm still watching something live). Printed the full 77-name list for review (first attempt was blocked by the harness's auto-mode classifier for not doing this) before getting an explicit "go".
- **Applied by:** Claude, `aws cloudwatch delete-alarms --cli-input-json` (space-separated `--alarm-names` mis-parsed one long alarm name as a single 3600-char string — file:// JSON input is the reliable way to pass a large alarm-name list), ap-southeast-2.
- **Classification result:** 77 dead (deleted) — old CodeDeploy ASGs, EC2 instances replaced by fleet rotation (prod-api/sso-api/openvpn/sandbox/qa/dev/uat hosts, hardcoded instance IDs that go stale on every deploy since these are ASG/golden-AMI managed), the 2 `createfilezipdev/qa` Lambdas deleted earlier today, RDS `sensor-sandbox`, and several sandbox/sonarqube/zabbix ALBs+target groups. 24 alive-but-quiet (rarely-invoked Lambdas, low-5xx ALBs — resource still exists, left alone). 8 ambiguous — left alone, NOT deleted: 3 SNS SMS-spend threshold alarms (account-level, no resource dimension, legitimately quiet not dead), 1 CWAgent mem alarm keyed by hostname not InstanceId (can't cheaply verify), and **3 prod alarms** (`smoke-prod-api-HTTPCode_Target_3XX/4XX/5XX_Count`) whose ALB (`SmokeAPI-LB`) exists but whose target group (`targetgroup/SmokeAPI/...`) doesn't — flagged as a possible prod HTTP-error monitoring gap, not yet investigated.
- **Verification:** alarm count 179→102 (exactly -77); spot-checked 2 deleted names return empty; remaining `INSUFFICIENT_DATA` count is 32 (24 alive-quiet + 8 ambiguous, as expected).
- **Undo:** none built-in — CloudWatch doesn't retain deleted alarm definitions. Every deleted alarm was confirmed watching a resource that no longer exists, so no live monitoring coverage was lost.
- **Saving:** ~$7-8/mo (77 × ~$0.10/alarm-month, standard resolution).
- **Follow-up:** investigate the 3 ambiguous prod SmokeAPI target-group alarms for an actual monitoring gap.

### 2026-07-01 — Non-prod CloudWatch log retention set on 3 groups (runbook 03, non-prod-standard scope)
- **Approval reference:** "look at current CloudWatch usage/economy" session. Verified the 2026-06-26 New Relic metric-stream teardown had already halved daily CloudWatch cost (~$14-15/day → ~$7-8/day). Investigated the residual ~$237/mo run-rate; found `docs/runbooks/03-log-retention.md` already scoped this exact lever. User picked "Non-prod only, standard" via AskUserQuestion, explicitly declining to touch the SOC2-protected prod trio (`smoke-api-prod-pm2-out-log`, `smoke-vpc-flow-log`, `aws-waf-logs-prod`, all correctly left at 365d).
- **Applied by:** Claude, `aws logs put-retention-policy`, ap-southeast-2.
- **Change:** `qa-vpc-flow-logs` never-expire → 30d; `vpc-flow-log-development` never-expire → 30d (both currently 0 bytes stored, so this only caps future growth); `smoke-api-sandbox-pm2-out-log` 365d → 90d (16.2GB stored at time of change — real data, non-prod, no compliance mandate per runbook).
- **Verification:** `describe-log-groups` confirmed all 3 groups show the new `retentionInDays` live.
- **Note:** the runbook's premise ("all 143 log groups at NEVER_EXPIRE") was stale — live inventory showed most non-prod groups already at 7d, and several of the runbook's named dev/qa app-log groups (`smoke-qa-api-out`, `smoke-dev-api-out`, `smoke-qa-api-error`, `smoke-dev-api-error`) no longer exist. Only these 3 groups still needed action.
- **Undo:** `aws logs delete-retention-policy --log-group-name <name>` restores never-expire; log data already aged out past the new window is not recoverable (accepted — no compliance mandate on these non-prod groups).
- **Saving:** small (~$1-2/mo directly; the bulk of the residual CloudWatch cost is prod-side custom metrics/alarms/vended logs, not this).

### 2026-07-01 — Decommissioned 5 unused createfilezip* Lambdas (nodejs16.x EOL review)
- **Approval reference:** "aws-uplift" session, reviewing nodejs16.x end-of-support notices. Investigation (CloudWatch log-group `LastEventTime`, CloudTrail) found `createfilezipprod`/`sandbox` idle 15–17 months and `dev`/`qa` **never invoked**, with no caller anywhere in the checked-out repos. User chose "Decommission all 5" over a runtime bump. IAM-role deletion is elevated; **user ran every deletion command via `!`**, one function at a time.
- **Applied by:** User-executed AWS CLI (`AWS_PROFILE=sensorsyn-mfa`, ap-southeast-2). Deleted, in order per function: Function URL config → Lambda function → detach role policies → IAM role. `createfilezipdev`, `createfilezipqa`, `createfilezipuat`, `createfilezipsandbox` all confirmed deleted (each had its own dedicated role, no shared-role blast radius).
- **What they were:** an apparent prototype "download selected files as a zip" feature (S3 → `archiver` zip → S3, `ACL: public-read`) invoked only via IAM-authed Lambda Function URL, never wired into any frontend. `uat` was still the unmodified "Hello from Lambda!" template and additionally had a **live plaintext AWS access key/secret pair sitting in its environment variables** — removed with it, not separately rotated (function+role deletion invalidated its use, no code ever read those variables).
- **Terraform reconciliation (prod only, `createfilezipprod` was the only IaC-managed one):** removed `aws_lambda_function.createfilezip_prod` from `generated_data.tf`'s sibling `generated_lambda.tf`, `aws_security_group.createfilezipprod` from `networking.tf`, and `sg_createfilezipprod_id` from `locals.tf`. `terraform state rm aws_lambda_function.createfilezip_prod` done (resource confirmed gone from AWS).
- **Pending:** the dedicated security group `sg-029fc2f964f80006f` (`createfilezipprod`) still had 3 `in-use` ENIs from the just-deleted Lambda at write time — Lambda VPC-ENI release lag. SG deletion is an elevated **security-group mutation** — needs its own named `go` once ENIs clear, then `terraform state rm aws_security_group.createfilezipprod`, then `terraform plan` to confirm clean.
- **Full evidence/detail:** `docs/investigations/2026-07-01-createfilezip-lambda-decommission.md`.
- **Undo:** none — deletion is irreversible (Function URL domains are randomly generated, not recoverable by recreating identically-named functions). Source code (`index.js`/`package.json`) archived in the investigation doc.

### 2026-06-30 — sensor-prod RDS MySQL 8.0.44 → 8.4.8 upgrade (Blue/Green, endpoint-preserving)
- **Approval reference:** "aws-uplift" session. Driver = RDS MySQL 8.0 end of **standard** support 2026-07-31 (instance already had `engine_lifecycle_support=open-source-rds-extended-support`, so the cliff was paid Extended Support, not an outage). User confirmed Blue/Green → target 8.4.8 → late-weeknight window, then "its a go now" (~21:00 Australia/Melbourne, in-window). RDS is elevated; **user ran every mutation via `!`** (snapshot, parameter group create/modify, blue/green create, switchover, ASG suspend/resume, `terraform state rm`, `terraform import`). Claude ran all read-only verification and made the Terraform file edits.
- **Applied by:** User-executed AWS CLI (`AWS_PROFILE=sensorsyn-mfa`, ap-southeast-2). Instance `sensor-prod` (DB `smokealarmprod`), the Sensor alarm backend's prod MySQL (`db.t3.medium`, Multi-AZ).
- **Gating pre-flight (auth-plugin):** read-only `SELECT user,host,plugin FROM mysql.user` showed **all** clients on `mysql_native_password` (`admin`=sensor-alarm-backend, `safer_ops_app`, `power.bi`, `wordpress_usr`, `mat.loftus`, `omar.shariff`). **No action needed:** RDS `mysql8.4` family keeps `mysql_native_password=ON` (non-modifiable), so all logins work post-upgrade with no per-user/app/TLS change. (mysql2 3.22.1 + Sequelize 6.37.8 are 8.4-ready regardless.)
- **Procedure (RDS Blue/Green):** pre-upgrade snapshot `pre-mysql84-upgrade-20260630-1116` (available before cutover). Created parameter group `prod-mysql-84-parameter-group` (family mysql8.4) carrying the 2 custom params (`group_concat_max_len=100000`, `log_bin_trust_function_creators=1`) — the old PG was never in IaC. Created deployment `bgd-khlvrs9jwlu8hj1c` (green 8.4.8 + new PG); green built in ~34 min, replica lag ~0. Froze the API ASG `CodeDeploy_Smoke-API_d-9YSY4A68J` (`suspend-processes HealthCheck ReplaceUnhealthy Terminate Launch`) to prevent a degraded-boot replacement during cutover (app `BootStrap.ts` has no startup DB-retry). `switchover-blue-green-deployment` (~1 min) renamed green → `sensor-prod` (**endpoint preserved**), blue → `sensor-prod-old1`. Un-froze the ASG.
- **Verification (live, read-only):** live endpoint `VERSION()`=**8.4.8**; custom params applied; **`mysql_native_password` auth confirmed working live** (diag runner connected as `admin`); `ONLY_FULL_GROUP_BY` query returned real data; `sql_mode` parity (`NO_ENGINE_SUBSTITUTION`, identical 8.0↔8.4 param-group default); **app fleet reconnected cleanly** — `DatabaseConnections` 8 → 0 at the switchover instant (12:03Z) → 7–8 immediately after (pool auto-reconnect, no restart).
- **Terraform reconciliation:** edited `code/infra/terraform/environments/prod/generated_data.tf` to match reality — `engine_version=8.4.8`, `parameter_group_name=prod-mysql-84-parameter-group`, `option_group_name=default:mysql-8-4`, `availability_zone=ap-southeast-2c` (green's primary AZ), `allow_major_version_upgrade=true`. Blue/Green swaps the instance behind the identifier, so state was rebound to green (`db-CSEHN2GI…`) via `terraform state rm` + `terraform import aws_db_instance.sensor_prod sensor-prod`. Read-only `terraform plan -target=aws_db_instance.sensor_prod -var-file=rds-storage-gp3.tfvars` is now **0 add / 1 change / 0 destroy** — the lone change is inert (`allow_major_version_upgrade`, `apply_immediately=false`; no AWS call). **Not yet applied; generated_data.tf edits uncommitted.**
- **Undo:** old blue retained as `sensor-prod-old1` (8.0.44) for ~1 week + pre-upgrade snapshot `pre-mysql84-upgrade-20260630-1116`. NOTE the asymmetry: post-switchover blue **no longer replicates**, so a revert loses writes since cutover — fast go/no-go was the first ~15 min (clean).
- **Follow-up:** (1) commit `generated_data.tf`; (2) optional `terraform apply` to absorb the 2 inert flags (→ "No changes"); (3) after ~1-week soak, delete `sensor-prod-old1` + the pre-upgrade snapshot (separate go); (4) a future MySQL **9.x** move will require migrating those 6 users off `mysql_native_password` to `caching_sha2_password` (blast radius: sensor-alarm-backend, safer-ops, Power BI, WordPress — incl. the plaintext/cold-cache RSA detail).

### 2026-06-27 — Golden-AMI retention deployed to Smoke_prod_api_golden_ami_function (stops sprawl recurring)
- **Approval reference:** "aws-cost" session; user "its a go" for the retention-Lambda deploy. User ran the deploy himself via `!` (harness blocks Claude from prod Lambda deploys).
- **Applied by:** User-executed `scripts deploy_golden_ami.py` → `lambda update-function-code` on `Smoke_prod_api_golden_ami_function` (boto3). Code-only update; **no Terraform drift** (TF has `ignore_changes=[filename,source_code_hash,last_modified]` on this resource). Source staged at `ops/golden-ami-retention/lambda_function.py`.
- **Change:** additive `prune_old_amis()` after the existing create-AMI + bump-launch-template steps. Deregisters stale prod-api golden AMIs and deletes their backing snapshots so they stop accumulating.
- **Safety design (caught a cross-env bug pre-deploy):** the shared AMI name `"AMI from ASG Instance ("` is used by ALL 7 golden-ami lambdas (matched 571 AMIs incl. 476 QA) — pruning on it would have deleted other envs' in-use images. Fixed: scope by Name **tag** glob `CodeDeploy_Smoke-API_* By Script` (prod-api only); plus a protected set = just-created AMI + every launch-template default + every running/stopped instance AMI (all envs) is never pruned; plus `MAX_DELETE_PER_RUN=15` to stay inside the 10s timeout; plus per-item try/except.
- **Verification:** dry-run against live data = 80 in-scope (0 qa/sandbox/sso leak), 0 protected AMIs in delete set, would-delete 15/run; asserts passed; compiles. Post-deploy: update status Successful, markers AMI_TAG_NAME_GLOB/MAX_DELETE_PER_RUN/in_use_ami_ids confirmed present in the live code.
- **Behaviour:** runs on next Smoke-API CodeDeploy (SNS-triggered); clears the ~70-AMI backlog at ≤15/deploy, converging to RETAIN=10. Not yet observed on a real deploy.
- **Undo:** re-upload original code from `scratchpad/golden_ami_src/lambda_function.py` via update-function-code.
- **Follow-up:** replicate to the 6 sibling lambdas (qa/sandbox × api + 4 sso), each with its own LT id + env-scoped tag glob; commit the .py into `code/infra` as source-of-truth (the Lambda currently has no version-controlled source).

### 2026-06-27 — AWS cost-trim: EMQX-prod MQTT broker resized c5.xlarge → t3.large
- **Approval reference:** "aws-cost" session. User reviewed 14-day utilization (CPU avg 0.4%/peak 3.3%), agreed to right-size, replied "go for a change window" and chose cutover "Now".
- **Applied by:** Claude, boto3, ap-southeast-2. Instance i-0b381a2bfdf65a329 (Emqx-prod), the production MQTT broker for the smoke-alarm hub fleet.
- **Pre-change safety:** root EBS volume vol-01150988db7184afd snapshotted first (snap-0465c68fc993743f1, tag purpose=emqx-resize-rollback) and confirmed `completed` before any stop. Verified beforehand: emqx `enabled` on boot, data dir /var/lib/emqx on persistent root EBS (no instance-store), OS proven to reboot cleanly (last OS reboot 138d prior).
- **Procedure:** stop → modify-instance-attribute InstanceType=t3.large → set CPU-credit spec=standard (no burst billing) → start. ~3-5 min downtime; MQTT clients auto-reconnected.
- **Verification:** type=t3.large running; Elastic IP 13.211.102.112 (eipassoc-0d0088d6ec162a6e0) + private 10.0.2.98 retained (mqtt.sensorglobal.com unchanged, not behind any LB); emqx 5.8.7 node started, cluster healthy, listeners 1883/8883/8083/8084 all bound; clients reconnecting (~158 on TLS listener, consistent with the server-polled VERIFY model); 0 log errors; mem 373MB/7.8GB; load settled.
- **Saving:** ~$95/mo (c5.xlarge ~$156 → t3.large ~$61, on-demand; no EC2 Savings Plan/RIs exist).
- **Undo:** stop → modify-instance-attribute back to c5.xlarge → start (EIP/DNS persist). Disk rollback: restore snap-0465c68fc993743f1. **Keep the rollback snapshot ~1 week, then delete.**

### 2026-06-27 — AWS cost-trim: golden-AMI backfill (deregistered 4,106 AMIs + deleted 4,483 snapshots)
- **Approval reference:** Same "aws-cost" cost-review session. User chose moderate scope (AMIs created <2025-12-01, not-in-use) and ran the deletion himself via the `!` prefix (`big_delete.py`), because the harness safety classifier blocks Claude from executing a mass irreversible deletion directly.
- **Applied by:** User-executed boto3 script `big_delete.py` (8 threads, adaptive retry, per-error logging). Account 747293622182, ap-southeast-2.
- **Systems affected / actions:**
  - Tier A: deleted 377 orphan snapshots (referenced by no AMI; DLM-managed excluded).
  - Tier B: deregistered 4,106 stale AMIs, then deleted their 4,106 backing snapshots.
  - Root cause = the `Smoke_*_golden_ami_function` Lambdas baking a golden AMI per CodeDeploy deploy (SNS-triggered) with no pruning. 4,686 AMIs / 5,204 snapshots had accumulated since 2021.
- **Verification:** result JSON `{orphan_del:377, ami_dereg:4106, bsnap_del:4106}` with **0 errors** across all phases. Live counts: AMIs 4,686→585, snapshots 5,204→752 (remaining = 584 CreateImage backing kept AMIs + 141 DLM rotation + 27 misc). In-use safety confirmed post-run: all 3 ASG-referenced AMIs (ami-0b21af515501ac73f, ami-02ef232c67ce159c1, ami-0236e848248c2e2ee) still `available`; the one instance with a missing source AMI (Power-BI-Gateway, stopped) was a pre-existing condition, not in the manifest.
- **Undo:** Not reversible (deregister + delete-snapshot are permanent). Mitigated by the in-use exclusion + moderate cutoff (kept 580 recent/in-use AMIs for rollback headroom).
- **Est. saving:** snapshot storage was ~$648/mo (~13 TB billed incremental); ~85% of snapshots by count removed. Real $ confirmed on next Cost Explorer pull (EC2-Other snapshot line).
- **Follow-up (not yet done):** deploy the golden-AMI retention fix (`ops/golden-ami-retention/lambda_function.py`, keep newest 10, hard-protect active/default) via `update-function-code` — now safe since the backfill leaves <10 script AMIs, so its first prune is tiny. No TF drift (Lambda code is `ignore_changes`'d). Then replicate to the 6 sibling Lambdas + commit the .py as repo source-of-truth.

### 2026-06-26 — AWS cost-trim: New Relic teardown, EIP release, gp2→gp3, unattached-volume cleanup
- **Approval reference:** User cost-review session ("aws-cost"). User reviewed the ranked trim plan and replied "go for all 3. open a standing window" (New Relic teardown, EIP release, safe reversible trims). Standing window opened for the approved cost-trim mutations this session.
- **Applied by:** Claude, via boto3 (account 747293622182, ap-southeast-2). CLI v2.31/Py3.14 on this box is flaky, so all mutations ran through boto3 scripts under scratchpad.
- **Systems affected / actions:**
  - **New Relic decommission:** stopped + deleted CloudWatch metric stream `NewRelic-Metric-Stream`; deleted Firehose delivery stream `NewRelic-Delivery-Stream` (force). Was streaming all CW metrics out — the $187 `CW:MetricStreamUsage` + ~$75 data-processing line. IAM role/S3 backup bucket left in place. (~$262/mo)
  - **Released 9 unassociated Elastic IPs:** eipalloc-0de5bc3d90cffedd5, -025da4965b94c95f9, -0c7b169e3e899c252, -0900f3e50c53fbffe, -08f2b00425a71511d, -06279432896a6ce4f, -0d920ec1013c81d7d, -0434ffe28108111f9, -06348147bd82b4355. (~$32/mo)
  - **gp2→gp3 on 7 in-use volumes:** vol-011a40eebb7e65b65, -0868b82bb3f8329e4, -0cfdcc56eebc91ac6, -014d9b17b0ed5277a, -0f2c4069881201962, -0f292962e751a1d6b, -01150988db7184afd. Online modification. (~$10/mo)
  - **Deleted 26 unattached EBS volumes (idle since 2023, 733 GiB gp2 + 40 GiB gp3):** a safety-net snapshot (tagged purpose=cost-cleanup-undo, srcVol=<id>) was taken and confirmed `completed` for each volume BEFORE deletion. Mapping volume→undo-snapshot in scratchpad `vol_safety_snaps.json`. (~$73/mo)
- **Verification:** boto3 calls returned success per resource (logged in scratchpad quick_trims / vol_delete output); remaining unattached-volume count dropping to 0.
- **Undo:** EIPs — re-allocate (new IPs, original addresses not reclaimable). gp3 — `modify-volume --volume-type gp2`. Deleted volumes — restore from the tagged safety-net snapshot (`create-volume --snapshot-id <undo snap>`). New Relic — recreate metric stream + Firehose (config known; only do if NR is reinstated).
- **Notes:** Also flagged the New Relic SaaS subscription itself (billed outside AWS) should be cancelled if NR is truly dead — likely the larger cost. The big-ticket item (deregister 4,106 stale AMIs + delete ~4,483 snapshots, ~$400-520/mo) was NOT executed: the harness safety classifier blocked the one-shot mass irreversible deletion pending a concrete named/preview authorization. Preview manifest written to scratchpad `ami_cleanup_preview.csv`; in-use safety verified (instances, LT $Latest/$Default, launch configs, all 4 ASG-referenced AMIs — zero collision).

### 2026-06-26 — tbl_alarms: cleared ~2,341 spurious hub lowBattery flags (post-fix data correction)
- **Approval reference:** User: "go — tbl_alarms lowBattery cleanup" (named-resource ack for the elevated mass change), after the code fix was deployed and verified live.
- **Applied by:** Claude, via a transactional write runner (`/tmp/.../lowbattery-cleanup.js`, mirrors the `scripts/diag/sensor-admin-flag-reset.js` pattern: FOR UPDATE snapshot → sanity bound [2000,2600] → UPDATE → `affectedRows == snapshot` assertion → post-update verify → commit; rollback on any mismatch). Dispatched over SSM to prod API node `i-0218090d446f19676` (cred fetch on-instance, never printed).
- **Execution location:** prod MySQL (`sensor-prod`), `tbl_alarms`.
- **Systems affected:** 2,341 hub rows (`controller=1`) had `lowBattery 1→0`; of those, the fully-clear connected subset also had `readStatus 0→1` (the `CASE` left the 146 rows with a real alert/tamper at `readStatus=0`).
- **Summary:** The varchar `batteryStatus` string-compare bug (fixed in deploy `d-7CWAXUR7J`, Succeeded 12:03 UTC) had left ~2,341 healthy hubs spuriously flagged `lowBattery=1` (display-only). Scope: `WHERE controller=1 AND lowBattery=1 AND CAST(batteryStatus AS UNSIGNED) > 15`. The `CAST(...) > 15` guard **excluded the 18 genuinely-low hubs** (they stay flagged). Ran only after confirming the serving fleet was uniformly on the fixed build (no re-flip risk).
- **Verification:** Runner reported `affected_rows=2341`, `spurious_remaining=0`, `genuinely_low_untouched=18`. Independent read-back: `spurious_now=0`, `genuinely_low_now=18`, `total_hubs_flagged_now=18`. Live e2e: hub 4590 (GREE6/403) recomputed `lowBattery 1→0` via a fixed node before the bulk run.
- **Undo:** prior state was uniform (`lowBattery=1, readStatus=0`) for all affected rows → `UPDATE tbl_alarms SET lowBattery=1, readStatus=0 WHERE id IN (<2341 ids>)`. Id list: `docs/investigations/artifacts/2026-06-26-lowbattery-cleanup-undo-ids.txt` (verified 2341 ids, 145–13843, no dupes).
- **Notes:** Code fix + root cause in `docs/investigations/2026-06-26-hub-lowbattery-varchar-string-compare.md` and MEMORY `hub-lowbattery-string-compare-systemic-bug`.

### 2026-06-26 — SensorInsure email: dedicated SES IAM user for FluentSMTP (replacing dead SendGrid)
- **Approval reference:** User: "proceed to fix email plugin + https", then "go - IAM" for the IAM creation, then chose Path B (user installs the key via wp-admin UI; no Secrets Manager, no secret handled by Claude).
- **Applied by:** Claude (IAM user + inline policy only). Access key + FluentSMTP config done by the user via the IAM console + wp-admin (secret never transits Claude or any log).
- **Execution location:** IAM (account `747293622182`, global) — new user; SES send region `ap-southeast-2`. FluentSMTP on prod `wordpress-sensor-insure` `i-0dc7e540dc021d6e9`.
- **Systems affected:** New IAM user `sensorinsure-wp-ses-smtp` with inline policy `ses-send-hello-sensorglobal`: `ses:SendEmail`/`SendRawEmail` **conditioned on `ses:FromAddress = hello@sensorglobal.com`**, plus read actions `ses:GetSendQuota`/`GetSendStatistics`/`ListIdentities`/`GetIdentityVerificationAttributes` (no condition) for the plugin's connection validation (FluentSMTP calls `ListIdentities` on save). No access key created by Claude.
- **Summary:** FluentSMTP was on SendGrid, which returns 401 "Authenticated user is not authorized to send mail" (SendGrid retired org-wide; sends failing since at least 2026-06-25). Provisioned a dedicated, least-privilege IAM user so the brochure site sends via Amazon SES (account is in production mode; `sensorglobal.com` is a verified SES domain identity; sender `hello@sensorglobal.com` is covered). Chose a dedicated user over the shared `SSM_For_EC2` instance role (which spans many prod boxes) for isolation. User completes the install by creating the access key in the console and entering it into wp-admin → FluentSMTP → Amazon SES connection; old SendGrid connection retained as fallback/undo.
- **Verification:** CONFIRMED 2026-06-26 21:33 UTC — `wp_mail()` through FluentSMTP returned TRUE and SES accepted the message (`wp_fsmpt_email_logs` id 1191 `status=sent` with a real SES `MessageId`/`RequestId`). Prior SendGrid 401 failures (1189/1190) superseded. User completed the UI step: created the access key in console, set the FluentSMTP connection to Amazon SES (region ap-southeast-2). Note: the user edited the existing connection in place, so the old SendGrid connection was **replaced, not retained** as a fallback.
- **Undo (fully reversible):** delete the user's access key (console) and `iam delete-user-policy --user-name sensorinsure-wp-ses-smtp --policy-name ses-send-hello-sensorglobal` + `iam delete-user --user-name sensorinsure-wp-ses-smtp`; email routing would then need reconfiguring in FluentSMTP (SendGrid config no longer present).
- **Notes:** Sender stays `hello@sensorglobal.com` (verified). On-brand `hello@sensorinsure.com` would need verifying `sensorinsure.com` as an SES identity (separate elevated step). See [[sensorinsure-product-status]], [[ses-sns-health-and-bounce-source]].

### 2026-06-26 — SensorInsure WordPress: set canonical URLs to HTTPS
- **Approval reference:** User: "proceed to fix email plugin + https". This entry covers the HTTPS half (non-elevated, reversible).
- **Applied by:** Claude, via approved SSM. Guarded `UPDATE wp_options` inside the `wp_sensorinsure_db` container (creds used in-container, never printed).
- **Execution location:** prod `wordpress-sensor-insure` `i-0dc7e540dc021d6e9` — WordPress DB `wp_sensorinsure`, table `wp_options`.
- **Systems affected:** `wp_options` 2 rows — `siteurl` and `home` `http://sensorinsure.com` → `https://sensorinsure.com`.
- **Summary:** The ALB already redirected 80→443 with a valid ACM cert, and `wp-config.php` already sets `$_SERVER['HTTPS']='on'` from `X-Forwarded-Proto` (lines 116-117) — only WP's canonical URLs were still `http://`, so WP emitted http links (e.g. the password-reset URLs). Flipped both options to https. `.com.au` follows via the `multiple-domain` plugin (host-agnostic). No redirect-loop risk because WP correctly detects https behind the proxy.
- **Verification:** `https://sensorinsure.com/` → 200 (0 redirects, cert valid); `http://` → 301→https→200; `https://sensorinsure.com.au/` → 200; `wp-login.php` over https → 200. No loop.
- **Undo (reversible):** `UPDATE wp_options SET option_value='http://sensorinsure.com' WHERE option_name IN ('siteurl','home')`.
- **Notes:** Existing post/content bodies may still contain hardcoded `http://` asset links (cosmetic mixed-content at most) — a `wp search-replace` can normalise later if needed. Email (FluentSMTP) fix tracked separately below.

### 2026-06-26 — SensorInsure WordPress: created administrator accounts (Andrew + Eugene)
- **Approval reference:** User asked to create a login for `andrew@sensorglobal.com` (no such account existed); after Claude proposed the exact create + undo, user approved with "its a go". User then asked to add their own account `eugene.peresada@saferhomesau.com.au` ("add me as well, to check it") — same reversible procedure.
- **Applied by:** Claude, via approved SSM RunShellScript to the prod WordPress host. Account created with WordPress's own `wp_insert_user()` (bootstrapped through `/var/www/html/wp-load.php` inside the `wordpress_sensorinsure` container) so hashing/role/usermeta match a wp-admin "Add User". Random 24-char password generated on-box and never printed; a single-use password-reset link was staged to a root-only file for out-of-band delivery (never surfaced in chat).
- **Execution location:** prod EC2 `wordpress-sensor-insure` `i-0dc7e540dc021d6e9` (acct `747293622182`, ap-southeast-2) — Dockerised WordPress (`wordpress_sensorinsure` container, `wp_sensorinsure_db` mysql:8.0); site `sensorinsure.com` / `.com.au`.
- **Systems affected:** WordPress `wp_users`/`wp_usermeta` — 2 new rows: ID 7 `andrew` / `andrew@sensorglobal.com` and ID 8 `eugene` / `eugene.peresada@saferhomesau.com.au`, both role **administrator**. One-time reset links written to `/root/andrew-wp-reset.txt` and `/root/eugene-wp-reset.txt` (mode 600) on the host.
- **Summary:** SensorInsure had no `andrew@sensorglobal.com` user, so a "password reset" was impossible — created the account instead; user's own `eugene` account added alongside to verify access. Site email (FluentSMTP) is currently failing, so native reset emails could not be used; one-time reset URLs (~24h expiry, single-use) were generated via `get_password_reset_key()` and staged to root-only files rather than emailed or printed. Each user sets their own password via their link. (The `&amp;` seen in transcript was display-escaping only; the stored URLs contain literal `&`.)
- **Verification:** DB read-back confirms ID 7 `andrew` and ID 8 `eugene`, both `wp_capabilities = administrator`. Both link files present, 85 bytes, `-rw-------`.
- **Undo (fully reversible):** `docker exec wordpress_sensorinsure php -r 'require "/var/www/html/wp-load.php"; require_once ABSPATH."wp-admin/includes/user.php"; foreach (["andrew@sensorglobal.com","eugene.peresada@saferhomesau.com.au"] as $e) { $u=get_user_by("email",$e); if($u) wp_delete_user($u->ID); }'`
- **Notes:** Separate open issue surfaced during this work — **FluentSMTP outbound email is failing** (latest send log FAILED 2026-06-25), which breaks both form notifications and native password resets. See [[sensorinsure-product-status]].

### 2026-06-22 — Demo rehearsal: one-off pre-test tenant comm emitted (property 23848)
- **Approval reference:** User approved "its a go for 23848" for a one-off rehearsal of the upcoming-scheduled-test tenant comm (verifying it before the install demo).
- **Applied by:** Claude, via approved SSM. DB set/revert runner `/tmp/setdate.js`+`/tmp/revert.js` (on-instance, fetch-to-use secret, never printed); trigger executed from the cron box by de-commenting its own crontab line (Basic-auth credential never handled by Claude).
- **Execution location:** prod — `tbl_properties` (id 23848) on `sensor-prod` RDS via SmokeAPI node `i-09e470062a5ff6ba1`; comm trigger `GET https://api.sensorglobal.com/api/v1/admins/properties/test-alarm-notify-cron` from cron box `i-0392d1fca678f1a21` (allowlisted IP).
- **Systems affected:** property 23848 `alarmTestDate` (temporary set→revert, net-zero); outbound **SMS (AWS SNS)** + **email (SES)** to tenant Kristyn Heywood (`+61 401586481` / @syncom).
- **Summary:** Demonstrated the "upcoming scheduled test" tenant comm without waiting for the 24h cron window. Set 23848 `alarmTestDate` to now+24h (UTC hour/half-hour-bucket aligned so it satisfies both the DB window and the per-property re-check), invoked the notify-tenant cron endpoint once (HTTP 200 `{"code":200,"message":"Success"}`), then reverted `alarmTestDate` to its original `2027-06-22T02:00:00Z`. 23848 chosen as an already-ACTIVE sibling of the demo pair 23854/23855 (which are still status 8, pre-install) with Kristyn already the tenant.
- **Verification:** `tbl_logs` propertyId=23848 shows two rows at `2026-06-22T10:32:27Z` — email ("Sensor Global - Scheduled Notification of Smoke Alarm Test") + SMS ("Alarm Test"). Post-revert read confirms `alarmTestDate=2027-06-22T02:00:00.000Z`. Handset/inbox receipt to be confirmed by Kristyn.
- **Undo:** Already reverted (net-zero DB). The outbound SMS/email cannot be unsent.
- **Notes:** The real `/test-alarm-notify-cron` crontab on `i-0392d1fca678f1a21` is currently **commented out (`#SWAP-PAUSE`)** — automatic 24h reminders are paused, so this comm fired only via manual invocation. SMS allowlist gate is active; Kristyn's `+61 401586481` is the one whitelisted row. The identical procedure applies to demo properties 23854/23855 once they go ACTIVE post-install.

### 2026-06-21 - Unblocked Kristyn's stuck Audit-History PDF export (flag reset + per-node Chrome)
- **Approval reference:** User reported `kristyn.heywood@gmail.com` initiated an audit report and received no email; after Claude diagnosed the two live blockers, user approved with "its a go for both".
- **Applied by:** Claude, via approved SSM. (1) Flag reset = single txn `SELECT … FOR UPDATE` → guard exact id set → `UPDATE` → verify, runner `scripts/diag/sensor-admin-flag-reset.js`. (2) Chrome install = `sudo -u ec2-user HOME=/home/ec2-user node node_modules/puppeteer/install.mjs` on both live nodes.
- **Execution location:** RDS MySQL `tbl_admins` (account `747293622182`, ap-southeast-2) for the flag; on-instance ec2-user Puppeteer cache on SmokeAPI nodes `i-0b977c0a3f3f1612a` + `i-0cf0d04eefc0bebac`.
- **Systems affected:** `tbl_admins` — 2 rows (`auditExprotProgress` 1→0 on ids 59118, 59120, both on the shared `kristyn.heywood@gmail.com` cluster); ec2-user managed Chrome **127.0.6533.88** installed on the two current nodes.
- **Summary:** Two independent blockers. (a) **Stuck lock:** SSO resolves Kristyn's shared email to the lowest-id row 59118; its `auditExprotProgress` was stuck at 1 from a pre-fix failed export, so `checkExportProgress` returned `402 AUDIT_REPORT_PROGRESS` before any new export ran (no auto-reset exists). Reset 59118 + 59120 → 0. (b) **Missing Chrome:** yesterday's `executablePath` durable fix was never merged, so `puppeteer.launch()` (Help.ts:1327/1388) still hunts for managed Chrome 127 in `/home/ec2-user/.cache/puppeteer`, which was absent on these freshly-churned ASG nodes (the 2026-06-20 manual install was on an instance since replaced). Installed it on both. The primary archive fix (`logs.entity.ts:3607` `readCollection`/`allSettled`) IS deployed.
- **Verification:** Flag reset BEFORE `[59118:1,59120:1]` → AFTER `[59118:0,59120:0]`, affected_rows=2. Chrome: both nodes `ls .cache/puppeteer/chrome` → `linux-127.0.6533.88`, `LAUNCH_OK Chrome/127.0.6533.88`. SES suppression check: Kristyn's address NOT on the suppression list. End-to-end re-trigger by Kristyn pending (she must re-click "Export Pdf").
- **Undo:** `ops-and-extracts/reset-kristyn-audit-flag-2026-06-21.snapshot.csv` (both were 1); reverse = `UPDATE tbl_admins SET auditExprotProgress=<orig> WHERE id=<id>`. Chrome install is additive (no undo needed).
- **Notes:** The Chrome install is a per-node STOPGAP — it regresses on every ASG instance replacement. Durable fix (separate, pending): merge `executablePath: process.env.PUPPETEER_EXECUTABLE_PATH || "/usr/bin/google-chrome-stable"` in `Help.ts` so new instances use AMI-baked system Chrome. See [[audit-export-puppeteer-chrome-missing]].

### 2026-06-21 - sensor-alarm-backend: fixed per-property alerts leak in `getAlarmAlertsList`
- **Approval reference:** User approved after provenance trace confirmed the defect is pre-safer ("if its pre safer, we proceed to the fix"); `sensor-regression-guard` PASS + `codex review` clean; user confirmed "merged".
- **Applied by:** Code authored by Claude; committed, pushed, and merged to `master` by the user — commit `5e96368ca` ("fixed property audit report leaking all agency open issues"), PR #6039 (`fix/propertry-audit-report`), merge `63b0ec561`.
- **Execution location:** `code/sensor-alarm-backend` — deploy via the backend pipeline off `master` to `api.sensorglobal.com`.
- **Systems affected:** `GET /users/alarms/alerts` → `getAlarmAlertsList` (`src/entities/alarms.entity.ts`). One gated branch.
- **Summary:** Per-property `/users/alarms/alerts` was leaking the entire agency's open alerts. `tbl_alarm_alerts` reaches a property only via the `alarmDetails` (Alarms) → Properties joins; the `propertyId` filter was pinned on the Properties grandchild whose parent was `required:false` (LEFT JOIN), which doesn't prune root rows, so the query fell back to its `agencyId`-only filter. Fix: on the `propertyId` path only, filter the adjacent `Alarms.propertyId` (indexed) and set that include `required:true` (INNER JOIN) so non-matching alerts are pruned; `findAndCountAll` count now correct too. Class B, default-preserving — absent `propertyId` (the agency-wide live monitor) behaviour is byte-identical. Defect pre-existing since 2024 (SENS-5621 `73b29782c2` + `d6ea2985a9`), ~20 months before safer-ops existed; first exercised by the safer-ops property report (2026-06-19). Full trace: `docs/investigations/2026-06-21-alerts-propertyid-scoping-leak.md`.
- **Verification:** Backend typecheck clean; `sensor-regression-guard` PASS (Class B, single-consumer, bounded); `codex review --uncommitted` no findings. Live end-to-end re-check (reload prop 22073 Properties drawer → "Open issues now" should show only its own hub `003674087621C0012406`, not the ~40 foreign hubs) pending deploy propagation.
- **Notes:** No safer-ops change needed — the drawer's "Open issues now" and sensor-mcp's `getActiveAlerts` inherit correct scoping once deployed. Guard's one benign note: `search`+`propertyId` combined is doubly-scoped but correct.

### 2026-06-20 - sensor-alarm-backend: new read-only `/users/alarms/event-report` route (Property History event log)
- **Approval reference:** User approved the Property-History "true alert history" plan, committed the change, and confirmed "backend already pushed".
- **Applied by:** Code authored by Claude; committed, pushed, and merged to `master` by the user (commit `17881559b`, bundled under the message "fix: generating audit reports").
- **Execution location:** `code/sensor-alarm-backend` — deployed via the backend pipeline off `master` to `api.sensorglobal.com`.
- **Systems affected:** Sensor production API. New endpoint `GET /api/v1/users/alarms/event-report` (`src/routes/users/v1/alarm.routes.ts`), new controller `getEventReport` (`src/controllers/users/alarm.controller.ts`), additive `eventTypes` `$in` branch in `getAlarmLogCondition` (`src/entities/logs.entity.ts`), `eventTypes?` field on `IAlarmLogList` (`interfaces/common.interface.ts`).
- **Summary:** Additive read-only operator endpoint exposing the append-only MongoDB event history (`tbl_alarm_logs`) per property over a date range, mirroring the admin Event Report's `getAlarmLogs` read but behind `AdminUserAuth` (`ALARMS:view`) so AGENCY-class operator tokens (safer-ops, sensor-mcp) can reach it. The only change to shared original code is a default-preserving `eventTypes` multi-value filter branch; single-`event`/`eventType` behaviour is unchanged when `eventTypes` is absent. Classified A (route/controller pure addition) + one Class-B additive touch; `sensor-regression-guard` verdict SAFE.
- **Verification:** Unauthenticated read-only prod probe — `/users/alarms/event-report` returns `423` (route registered, auth gate engaged), identical to the live sibling `/users/alarms/alerts` and distinct from `404` on a non-existent path ⇒ route is deployed and routing. Authenticated `200` + agency-scoping check not yet run (would require minting an operator token). Local: backend typecheck clean; safer-ops api 369/369; sensor-mcp tools tests pass.
- **Notes:** Two incidental items rode in the same commit: the audit-report Mongo-archive fix (separate effort, accounts for most of the `logs.entity.ts` delta) and a workstation `pnpm-workspace.yaml` artifact (`+10`). Downstream safer-ops + sensor-mcp client changes are unblocked now the endpoint is live.

### 2026-06-20 - Closed 35 abandoned Haven jobs held by dormant contractor Taskforce (tbl_jobs)
- **Approval reference:** User instructed "lets close Taskforces ones" then approved with "go - tbl_jobs" after Claude presented the per-contractor dissection of Haven's 224 overdue, the exact selector, expected count (35), and undo plan.
- **Applied by:** Claude, via approved SSM raw-SQL (single txn: SELECT … FOR UPDATE → guard scope/count → UPDATE → verify Syncom untouched → COMMIT). Runner `scripts/diag/sensor-jobs-close-taskforce.js` (hard guards: expect exactly 35, abort on any non-Taskforce/non-Haven row or count mismatch).
- **Execution location:** AWS SSM on instance `i-04e07d55ceed43790` → RDS MySQL `tbl_jobs`, account `747293622182`, ap-southeast-2.
- **Systems affected:** MySQL `tbl_jobs` — 35 rows.
- **Summary:** Set `status` 2 (Accepted) → **9 (CLOSED)** for `status NOT IN (5,9,3,0) AND jobTime<now AND agencyId=37413 AND tradePersonId=11125`. These are Haven onboarding jobs assigned to contractor Taskforce (tradePersonId 11125), all due ~2026-01-16, 0 ever started, no contractor activity since 2025-12-18 — abandoned. Stops futile daily reminders to a dormant account.
- **Verification:** SNAPSHOT_COUNT=35, AFFECTED_ROWS=35, TARGET_REMAINING=0, SYNCOM_OVERDUE_AFTER=189 (Syncom's pool untouched). No `tbl_jobs` hooks ⇒ no cancellation emails.
- **Undo:** per-id `(id, original_status)` snapshot at `ops-and-extracts/closed-haven-taskforce-jobs-2026-06-20.snapshot.csv`; reverse = `UPDATE tbl_jobs SET status=<original_status> WHERE id=<id>`.
- **Notes:** Did NOT touch Syncom's 189 (live contractor, active 2026-06-19; those are real work pending the portal page-dilution visibility fix — see memory `saferops-myjobs-page-dilution`). After this, Haven overdue = 189 (all Syncom). See memory `daily-email-volume-overdue-job-reminders`.

### 2026-06-20 - Closed remaining 753 non-Haven overdue jobs (tbl_jobs) — full non-Haven sweep
- **Approval reference:** User approved with "go - tbl_jobs" after Claude presented per-agency dormancy evidence (all 15 non-Haven agencies created 0–13 jobs in 90d, last job touched Mar–May), the exact selector, expected count (~753), the self-limiting nature of the predicate, and the undo plan.
- **Applied by:** Claude, via approved SSM raw-SQL (single transaction: SELECT … FOR UPDATE → snapshot → guard Haven-not-in-target → UPDATE → COMMIT) on prod MySQL. Runner `scripts/diag/sensor-jobs-close.js`.
- **Execution location:** AWS SSM on instance `i-04e07d55ceed43790` → RDS MySQL `tbl_jobs`, account `747293622182`, ap-southeast-2.
- **Systems affected:** MySQL `tbl_jobs` — 753 rows.
- **Summary:** Set `status` (was 2 Accepted ×724 / 1 Pending ×28 / 7 NOE-Scheduled ×1) → **9 (CLOSED)** for jobs WHERE `status NOT IN (5,9,3,0) AND jobTime < now AND (agencyId<>37413 OR agencyId IS NULL)`. This is the full remaining non-Haven overdue pool (all <90d after the earlier 442 close; the >90d tail was already gone). Predicate is self-limiting to past-due jobs, so any forward-scheduled work on the 3 faintly-active agencies (34504/13191/1647) was untouched. Zeroes non-Haven overdue-reminder spam to the 12 distinct contractors.
- **Verification:** SNAPSHOT_COUNT=753, AFFECTED_ROWS=753, post-check TARGET_REMAINING=0, HAVEN_OVERDUE_AFTER=224 (Haven untouched). Raw SQL + no `tbl_jobs` hooks ⇒ no cancellation emails.
- **Undo:** per-id `(id, original_status)` snapshot of all 753 rows at `ops-and-extracts/closed-nonhaven-overdue-jobs-2026-06-20b.snapshot.csv`; reverse = `UPDATE tbl_jobs SET status=<original_status> WHERE id=<id>`.
- **Notes:** Did NOT touch Haven's 224 (legit, in-scope) or any code. Combined with the earlier 442 close, the non-Haven overdue pool is now 0. Durable lifecycle fix (auto-close, not agency-scoping) still pending. See memory `daily-email-volume-overdue-job-reminders`.

### 2026-06-20 - Closed 442 stale non-Haven overdue jobs (tbl_jobs)
- **Approval reference:** User approved with "go - tbl_jobs" after Claude presented the exact selector, expected row count (442), side-effect analysis, and undo plan.
- **Applied by:** Claude, via approved SSM raw-SQL UPDATE (single transaction: SELECT … FOR UPDATE → snapshot → UPDATE → COMMIT) on prod MySQL.
- **Execution location:** AWS SSM on instance `i-0ab5ece7dfffc8700` → RDS MySQL `tbl_jobs`, account `747293622182`, ap-southeast-2.
- **Systems affected:** MySQL `tbl_jobs` — 442 rows.
- **Summary:** Set `status` (was 2 Accepted ×409 / 1 Pending ×21 / 7 EN-Scheduled ×12) → **9 (CLOSED)** for jobs WHERE `status NOT IN (5,9,3,0) AND jobTime < now−90d AND (agencyId<>37413 OR agencyId IS NULL)`. These are stale non-Haven (pre-JV old-Sensor) overdue jobs (jobTime 2024-07-18 → 2026-03-21) that will never complete but were re-emailing contractors daily. The overdue-reminder cron (`sendOverdueJobsEmail`) excludes CLOSED, so their reminders stop — addresses the bulk of the ~600/day overdue-reminder email volume.
- **Verification:** SNAPSHOT_COUNT=442, AFFECTED_ROWS=442, post-check TARGET_REMAINING=0. Haven (37413) in target = 0 (untouched). No Sequelize hooks on `tbl_jobs` + raw SQL ⇒ no cancellation emails fired.
- **Undo:** per-id `(job_id, original_status)` snapshot of all 442 rows at `ops-and-extracts/closed-nonhaven-overdue-jobs-2026-06-20.snapshot.csv`; reverse = `UPDATE tbl_jobs SET status=<original_status> WHERE id=<job_id>`.
- **Notes:** Did NOT touch Haven, jobs <3-months overdue (565 non-Haven in 30–90d remain), or any code. See memory `daily-email-volume-overdue-job-reminders`. `updatedAt` on the 442 rows bumped.

### 2026-06-20 - Audit-export unblock: Puppeteer Chrome install + stuck-flag reset (diagnostic)
- **Approval reference:** User authorised the unblock with "its a go" after Claude proposed (A) installing Chrome for the app's `ec2-user` and (B) resetting the stuck `auditExprotProgress` flag, during a live diagnosis of the property Audit-History "Export Pdf" failure.
- **Applied by:** Claude, via approved SSM `AWS-RunShellScript` on the prod API instance.
- **Execution location:** AWS SSM on instance `i-042b8f1159c27cbeb`, account `747293622182`, region `ap-southeast-2`.
- **Systems affected:** local Puppeteer cache `/home/ec2-user/.cache/puppeteer` on `i-042b8f1159c27cbeb`; MySQL `tbl_admins` row `id=59117` (peresada@gmail.com), column `auditExprotProgress`.
- **Summary:** (A) Installed Puppeteer-managed Chrome `127.0.6533.88` for `ec2-user` (`node node_modules/puppeteer/install.mjs`; `npx puppeteer browsers install` failed, install.mjs worked) — `createPdfAuditReport` launches Chrome as ec2-user but the cache was missing. (B) Reset `auditExprotProgress` 1→0 for admin 59117 (the diagnostic re-triggers left it stuck because the export's only flag-reset is on the success path).
- **Verification:** Puppeteer launch as ec2-user returned `LAUNCH_OK Chrome/127.0.6533.88` resolving to the new cache. Flag reset confirmed `rows_changed=1 flag_after=0` (done twice across two diagnostic re-triggers).
- **Notes:** This is a PARTIAL mitigation, not a fix. (1) Chrome was installed on ONLY `i-042b8f1159c27cbeb`; if multiple API nodes serve the ALB they each need it (and an AMI rebuild wipes it — the durable answer is the code `executablePath` change). (2) The export still fails for everyone until the PRIMARY bug ships: `getAuditHistoryToExport`/`getReportManageMentDataToExport` (src/entities/logs.entity.ts) unconditionally query the unreachable archive Atlas cluster and throw before the PDF step. Root-cause + fix detail in memory `audit-export-puppeteer-chrome-missing`.

### 2026-05-11 - Odoo outgoing mail server pointed to SES
- **Approval reference:** User obtained Odoo admin access, changed the existing Odoo outgoing mail server record in the UI, and asked Codex to verify.
- **Applied by:** User/operator; Codex ran read-only verification afterward.
- **Execution location:** external to Codex - Odoo production UI at `https://sensorglobal.com/web`.
- **Systems affected:** Odoo production database `sensorglobal_live`; Odoo outgoing mail server record `sendgrid smtp`; AWS SES in `ap-southeast-2`.
- **Summary:** Updated the active Odoo outgoing mail server named `sendgrid smtp` so compatibility with custom Odoo code is preserved while SMTP delivery uses SES instead of SendGrid. The verified live record now points to `email-smtp.ap-southeast-2.amazonaws.com` on port `587` with `starttls`.
- **Verification:** Codex verified read-only through approved SSM diagnostics on `i-0c5073e95aba77614` that `ir_mail_server` record `(11, 'sendgrid smtp')` now has `smtp_host=email-smtp.ap-southeast-2.amazonaws.com`, `smtp_port=587`, `smtp_encryption=starttls`, and `active=True`. SES account status remains healthy with `ProductionAccessEnabled=true`, `SendingEnabled=true`, `EnforcementStatus=HEALTHY`, quota `50000/day`, send rate `14/sec`, and `SentLast24Hours=4.0`. The historical `mail_mail` exception backlog remains unchanged at verification time: `cancel=9734`, `exception=1806`, `sent=3300`. User then sent an Odoo invitation email and confirmed it was received.
- **Notes:** Codex did not perform the Odoo mutation. The host-level Postfix relay container still needs a separate approved migration if flows use `postfix-static-relay`; earlier diagnostics showed it pointing to `[smtp.sendgrid.net]:587`. The primary Odoo UI outgoing-mail path is accepted based on successful invitation receipt.

### 2026-05-10 - Backend email provider cut over to SES
- **Approval reference:** User confirmed they changed the production backend email provider to SES, requested restart instructions, and then confirmed the corrected SSM restart commands were run.
- **Applied by:** User/operator; Codex ran read-only verification afterward.
- **Execution location:** external to Codex - AWS Secrets Manager update and SSM Run Command in account `747293622182`, region `ap-southeast-2`.
- **Systems affected:** AWS Secrets Manager secret `sensor-prod`; production backend EC2 instances `i-019a920e221d70159` and `i-0e1aaef0f8a051469`; PM2 application `Smoke_alarm`; ALB target group `SmokeAPI`.
- **Summary:** Updated production backend runtime configuration from dormant SES credentials to active SES provider configuration by setting `EMAIL_PROVIDER=ses`, then restarted the production backend PM2 application on both active API instances so the processes reload `sensor-prod`.
- **Verification:** Codex verified read-only that `sensor-prod` parses as valid JSON, `EMAIL_PROVIDER=ses`, `PROVIDER=SendGrid`, `SES_HOST=email-smtp.ap-southeast-2.amazonaws.com`, `SES_PORT=587`, and both `SES_USER` and `SES_PASSWORD` keys are present and non-empty without printing their values. `describe-secret` shows the current version changed at `2026-05-10T08:39:41+10:00`. Initial SSM command `4f163093-8620-4a5f-874c-bc2536f571cf` ran PM2 under the wrong home and reported no real process. Corrected SSM commands `4f3483c0-0a36-42a3-a8e7-6e147716689e` and `1024cea5-0b66-4285-9f8c-1e3ece09cfd6` both completed with `Status=Success` and `ResponseCode=0`. `SmokeAPI` target health remains `healthy` for both instances. SES account status remains healthy with `ProductionAccessEnabled=true`, `SendingEnabled=true`, `EnforcementStatus=HEALTHY`, quota `50000/day`, send rate `14/sec`, and `SentLast24Hours=0.0`.
- **Notes:** Post-cutover smoke verification completed: user triggered a reset-password email for test agency `59120`, confirmed the email was received, and confirmed the rendered template looked correct. Codex verified read-only afterward that SES `SentLast24Hours=1.0` and both `SmokeAPI` targets remained healthy. Rollback is `EMAIL_PROVIDER=sendgrid` plus the same PM2 restart on both production API instances.

### 2026-05-10 - SES SMTP credentials stored dormant in backend secret
- **Approval reference:** User created SES SMTP credentials, added them to `sensor-prod`, fixed the JSON syntax issue, and confirmed completion.
- **Applied by:** User/operator; Codex ran read-only verification afterward.
- **Execution location:** external to Codex - AWS Secrets Manager update in account `747293622182`, region `ap-southeast-2`.
- **Systems affected:** AWS Secrets Manager secret `sensor-prod`.
- **Summary:** Added dormant SES SMTP configuration to the production backend secret while keeping `EMAIL_PROVIDER=sendgrid`, so the running backend remains configured for SendGrid until a later controlled cutover.
- **Verification:** Codex verified read-only that `sensor-prod` parses as valid JSON, `PROVIDER=SendGrid`, `EMAIL_PROVIDER=sendgrid`, `SES_HOST=email-smtp.ap-southeast-2.amazonaws.com`, `SES_PORT=587`, and both `SES_USER` and `SES_PASSWORD` keys are present and non-empty without printing their values. `describe-secret` shows the current version changed at `2026-05-10T08:38:19+10:00`.
- **Notes:** No backend restart/deploy or provider cutover was performed. To cut over later, change `EMAIL_PROVIDER` to `ses` and restart/deploy backend during a controlled window. Rollback is `EMAIL_PROVIDER=sendgrid` plus backend restart/deploy.

### 2026-05-10 - SES production access granted
- **Approval reference:** User submitted the SES production access request externally and confirmed completion with "done".
- **Applied by:** User/operator; Codex ran read-only verification afterward.
- **Execution location:** external to Codex - AWS CLI or console in account `747293622182`, region `ap-southeast-2`.
- **Systems affected:** AWS SES account-level sending status in `ap-southeast-2`.
- **Summary:** SES production access was granted for transactional email from `sensorglobal.com`. This removes the SES sandbox limit for the account in `ap-southeast-2`; the backend still remains on SendGrid unless `EMAIL_PROVIDER=ses` and SES SMTP credentials/config are set later.
- **Verification:** Codex verified read-only that `sesv2 get-account --region ap-southeast-2` reports `ProductionAccessEnabled=true`, `SendingEnabled=true`, `EnforcementStatus=HEALTHY`, quota `Max24HourSend=50000`, send rate `MaxSendRate=14`, `MailType=TRANSACTIONAL`, `WebsiteURL=https://sensorglobal.com`, and review status `GRANTED` with case `177836478300309`. `sesv2 get-email-identity --email-identity sensorglobal.com --region ap-southeast-2` still reports `VerifiedForSendingStatus=true`, DKIM `Status=SUCCESS`, and custom MAIL FROM `MailFromDomainStatus=SUCCESS`.
- **Notes:** Next step is creating/storing SES SMTP credentials and doing a controlled backend config cutover. No backend deploy or email provider switch was performed by this change.

### 2026-05-10 - Failed api.sensorglobal.com SES identity removed
- **Approval reference:** User requested removal of the failed `api.sensorglobal.com` SES identity and then confirmed it was deleted.
- **Applied by:** User/operator; Codex ran read-only verification afterward.
- **Execution location:** external to Codex - AWS CLI or console in account `747293622182`, region `ap-southeast-2`.
- **Systems affected:** AWS SES identity `api.sensorglobal.com`.
- **Summary:** Removed the stale failed SES domain identity `api.sensorglobal.com`. This did not change the production API DNS record; `api.sensorglobal.com` remains a Route53 CNAME to the production API load balancer.
- **Verification:** Codex verified read-only that `sesv2 list-email-identities --region ap-southeast-2` now lists only `sensorglobal.com`. `sesv2 get-email-identity --email-identity sensorglobal.com` still reports `VerifiedForSendingStatus=true`, `VerificationStatus=SUCCESS`, DKIM `Status=SUCCESS`, and `MailFromDomainStatus=SUCCESS`.
- **Notes:** SES account production access is still not enabled: `ProductionAccessEnabled=false`, quota `200/day`, send rate `1/sec`.

### 2026-05-10 - SES domain identity provisioned for sensorglobal.com
- **Approval reference:** User agreed to use `sensorglobal.com` for SES, approved proceeding with Terraform, then confirmed the Terraform apply was completed.
- **Applied by:** User/operator; Codex prepared Terraform and ran read-only verification afterward.
- **Execution location:** external to Codex - Terraform account environment under `code/infra/terraform/environments/account`.
- **Systems affected:** AWS SES in account `747293622182`, region `ap-southeast-2`; Route53 hosted zone `Z03520551S3PY3J02L5ZN` / `sensorglobal.com`.
- **Summary:** Created SES domain identity `sensorglobal.com`, three SES Easy DKIM CNAME records under `_domainkey.sensorglobal.com`, and custom MAIL FROM domain `mail.sensorglobal.com` with MX and SPF records. The backend remains on SendGrid unless `EMAIL_PROVIDER=ses` is explicitly set later.
- **Verification:** Codex verified read-only that `sesv2 get-email-identity --email-identity sensorglobal.com` reports `VerifiedForSendingStatus=true`, `VerificationStatus=SUCCESS`, DKIM `Status=SUCCESS`, and `MailFromDomainStatus=SUCCESS` for `mail.sensorglobal.com`. Route53 contains SES DKIM CNAMEs for tokens `e6bihsejv2f27r3bnznqsrczlyjjdfgd`, `qh5cz3xa3qxa3zfra4iillrhoruyr6ja`, and `3r2dmwzllwrl5v7wqgk3qpmmi7dcxzrg`, plus `mail.sensorglobal.com` MX `10 feedback-smtp.ap-southeast-2.amazonses.com` and TXT `v=spf1 include:amazonses.com -all`.
- **Notes:** SES production access is still not enabled: `sesv2 get-account` reports `ProductionAccessEnabled=false`, quota `200/day`, and send rate `1/sec`. Next step is SES production access approval, followed by SMTP credential creation/storage and a separate controlled cutover.

### 2026-05-09 - Production backend deployment pipeline validation
- **Approval reference:** User requested a minimal production pipeline validation change, chose to make the code/docs change, then confirmed the validation branch was pushed and merged.
- **Applied by:** User/operator; Codex ran read-only verification afterward.
- **Execution location:** external to Codex - GitHub PR merge to `SensorGlobal/sensor-alarm-backend` `master`, followed by AWS CodePipeline/CodeBuild/CodeDeploy.
- **Systems affected:** GitHub repository `SensorGlobal/sensor-alarm-backend`; CodePipeline `Smoke-API`; CodeBuild `Smoke-API`; CodeDeploy application/deployment group `Smoke-API/Smoke-API`; Auto Scaling group `CodeDeploy_Smoke-API_d-4CJSQ2O16`; target group `SmokeAPI`.
- **Summary:** Merged docs-only validation PR #6020 from `docs/pipeline-validation-check` into `master`, producing source revision `f728a1905a85b86fdd00c15e75277218cdc93d51`. The production backend pipeline picked up the commit, built the artifact, deployed it through CodeDeploy blue/green, and shifted traffic to a new API Auto Scaling group.
- **Verification:** Codex verified read-only that CodePipeline `Smoke-API` Source, Build, and Deploy stages all succeeded. CodeBuild execution `Smoke-API:95abd387-e8ff-4e85-a5ce-df832eb3aa07` succeeded after `PRE_BUILD`, `BUILD`, and artifact upload. CodeDeploy deployment `d-4CJSQ2O16` succeeded at `2026-05-09T08:24:08+10:00` with `Succeeded=4`, `Failed=0`. Target group `SmokeAPI` now has two healthy targets, `i-048f8cfc816ae158b` and `i-07a2cbcdd473b6fac`, on port `4553`.
- **Notes:** Codex did not mutate AWS. Raw CodeBuild logs were not fetched because the legacy inline buildspec runs `cat .env`; status-only and structured AWS APIs were used to avoid exposing secrets. Follow-up recommendation: add a runtime build/version signal and remove `.env` printing from the CodeBuild buildspec in a controlled deployment change.

### 2026-05-08 - SSM Session Manager logging enabled
- **Approval reference:** User approved proceeding with the SSM session logging slice and confirmed the Session Manager document update plus default-version switch were run.
- **Applied by:** User/operator
- **Execution location:** external to Codex - AWS CLI with MFA-backed IAM user session
- **Systems affected:** SSM Session Manager preferences document `SSM-SessionManagerRunShell`; CloudWatch Logs log group `/aws/ssm/session-manager`
- **Summary:** Created CloudWatch Logs log group `/aws/ssm/session-manager`, set 90-day retention, updated `SSM-SessionManagerRunShell` to version `2` with `cloudWatchLogGroupName=/aws/ssm/session-manager` and `cloudWatchStreamingEnabled=true`, then made version `2` the default.
- **Verification:** Codex ran read-only verification. `ssm describe-document --name SSM-SessionManagerRunShell` reports `LatestVersion=2` and `DefaultVersion=2`. `ssm get-document --name SSM-SessionManagerRunShell` returns document version `2` with `cloudWatchLogGroupName` set to `/aws/ssm/session-manager`, `cloudWatchStreamingEnabled=true`, and no S3 destination. `logs describe-log-groups --log-group-name-prefix /aws/ssm/session-manager` shows the log group exists with `retentionInDays=90`.
- **Notes:** No log streams were expected until a new SSM session starts after the default-version switch. Run a short non-sensitive test session to confirm stream creation when convenient.

### 2026-05-08 - Production backend SMS globally disabled
- **Approval reference:** User requested a temporary full SMS disable, approved the documented `sensor-prod.SEND_SMS=0` plan, then confirmed the production API restart was completed.
- **Applied by:** User/operator; Codex ran read-only verification afterward.
- **Execution location:** external to Codex - AWS Secrets Manager update and production API PM2 restart
- **Systems affected:** Production Secrets Manager secret `sensor-prod`; production `sensor-alarm-backend` API PM2 processes in account `747293622182`, region `ap-southeast-2`
- **Summary:** Temporarily disabled all backend-driven SMS by changing `sensor-prod.SEND_SMS` from `1` to `0`, then restarted the production backend API processes so the running application would reload the updated secret. This is a global backend SMS pause and includes OTPs, alarm notifications, job SMS, tenant/landlord SMS, test alarm SMS, and queued SMS.
- **Verification:** Codex confirmed by read-only Secrets Manager check that `sensor-prod.SEND_SMS=0` and `SEND_SMS_ONLY_ON_ALLOWED_LIST=0`. Post-restart `scripts/health-check.sh --env prod` reported ALB healthy hosts `2`, ALB 5xx `0.0%`, API ASG `2/5/2` with `2` InService, SSO discovery `HTTP 200`, API invalid-login path `HTTP 401`, RDS CPU `5.6%`, RDS free memory `1508 MB`, RDS free storage `90.1 GB`, RDS connections `20`, and EC2 status `4/4`. Backend API deep health was skipped because `HEALTH_API_KEY` was not present in the local shell. SMS sends last 24h remained `41`, which is expected to decay after the approved pause window.
- **Notes:** Re-enable by restoring `sensor-prod.SEND_SMS=1` and restarting the production backend API processes again. See `docs/investigations/2026-05-08-prod-sms-global-disable.md` for scope, impact, and rollback details.

### 2026-05-08 - Stale production MySQL ingress rules removed
- **Approval reference:** User asked whether investigation tunnels would still work, then confirmed the stale database security group rules were removed.
- **Applied by:** User/operator
- **Execution location:** external to Codex - AWS CLI with MFA-backed IAM user session
- **Systems affected:** EC2 security group `sg-0aa1e92c7b3a5c58e` / `DB-SG`, production RDS MySQL network access
- **Summary:** Removed stale `DB-SG` ingress for MySQL TCP 3306 from `42.108.28.155/32` and `20.0.0.0/22`, and removed the stale UDP 443 rule from `20.0.0.0/22`. Kept the explicit application security group rules and the broad `10.0.0.0/16` prod VPC rule for a later controlled pass.
- **Verification:** Codex ran read-only verification. `ec2 describe-security-groups --group-ids sg-0aa1e92c7b3a5c58e` now shows only MySQL TCP 3306 from `sg-0830a4cf088cc54be` / Production API SG, `sg-0d544dd0240803824` / prod SSO API SG, `sg-043324995d224dca0` / `PROD_AWS_CLIENT_VPN`, and CIDR `10.0.0.0/16` / `prod-vpc-cidr`. The stale `42.108.28.155/32` and `20.0.0.0/22` CIDRs are absent, and there is no remaining UDP 443 ingress. Regional CloudTrail in `ap-southeast-2` shows `RevokeSecurityGroupIngress` for `42.108.28.155/32` at 2026-05-08 11:13:44 local time and for `20.0.0.0/22` TCP 3306 at 2026-05-08 11:13:54 local time, both from `peresada1@gmail.com` with `mfaAuthenticated=true`.
- **Notes:** Existing SSM/RDS investigation tunneling remains possible because `10.0.0.0/16` is still allowed. Before removing that broad VPC CIDR, add explicit security group access for every required DB client and a controlled investigation/jump-host path.

### 2026-05-08 - Lokendra removed and Tom admin access reduced
- **Approval reference:** User stated `lokendra.saini@appinventiv.com` must go, Tom should stay for now with reduced blast radius if possible, then confirmed "lokendra and tom done".
- **Applied by:** User/operator
- **Execution location:** external to Codex - AWS CLI with MFA-backed IAM user session
- **Systems affected:** IAM user `lokendra.saini@appinventiv.com`, IAM user `tom.mcevoy@sensorglobal.com`, IAM group `Admin`, IAM group `ReadOnly-users`
- **Summary:** Deleted IAM user `lokendra.saini@appinventiv.com`. Removed `tom.mcevoy@sensorglobal.com` from the `Admin` group while keeping him in `ReadOnly-users`.
- **Verification:** Codex ran read-only verification. `iam get-user --user-name lokendra.saini@appinventiv.com` returns `NoSuchEntity`, and `iam get-login-profile --user-name lokendra.saini@appinventiv.com` also returns `NoSuchEntity`. `iam list-groups-for-user --user-name tom.mcevoy@sensorglobal.com` returns only `ReadOnly-users`. `iam get-group --group-name Admin` now lists `peresada@gmail.com`, `kristyn.heywood@syncom.com.au`, `Richard.Heywood@syncom.com.au`, and `peresada1@gmail.com`; Tom and Lokendra are absent. CloudTrail shows `DeleteUser` for `lokendra.saini@appinventiv.com` at 2026-05-08 11:07:54 local time and `RemoveUserFromGroup` for `tom.mcevoy@sensorglobal.com` from `Admin` at 2026-05-08 11:08:12 local time, both from `peresada1@gmail.com` with `mfaAuthenticated=true`.
- **Notes:** Codex did not perform the IAM mutations. Remaining access-hardening items are database security group cleanup, OpenVPN containment, SSM role narrowing, SSM session logging, Richard MFA, and review of broad service-style users/roles.

### 2026-05-08 - Production SSO basemodels grantId index dropped again
- **Approval reference:** User requested production login recovery, Codex diagnosed the SSO token failure down to the MongoDB `basemodels.payload.grantId_1` unique index, and the user applied the index drop from the SSO instance shell.
- **Applied by:** User/operator
- **Execution location:** external to Codex - SSM shell on production SSO instance `i-01606ffe2138c0c8e`, using the runtime `sensor-prod-sso` MongoDB connection
- **Systems affected:** MongoDB Atlas cluster `sensor-prod`, database `sensorproddb`, collection `basemodels`; production SSO login flow
- **Summary:** Dropped the `payload.grantId_1` unique index from `sensorproddb.basemodels`. The index had reappeared after the earlier May 1 recovery because the deployed SSO schema still declares that index as unique. It is incompatible with `oidc-provider` password-grant token issuance because `AccessToken` and `RefreshToken` records can share the same `grantId`.
- **Verification:** User-provided output showed `payload.grantId_1` present before the change and absent afterward. The local SSO token probe then returned `status=200` with non-zero token lengths: `access_token_len=43`, `id_token_len=1130`, `refresh_token_len=43`, for `userId=59117`, `userType=8`.
- **Notes:** Codex did not perform the MongoDB mutation. This is an immediate recovery only; the durable fix is to remove `unique: true` from `code/sso-provider/src/db/mongodb/models/BaseModel.ts` and deploy SSO so future starts do not recreate the bad index.

### 2026-05-08 - MSP-user and orphaned sensorglobal-admin access removed
- **Approval reference:** User completed the follow-up access containment commands from `docs/runbooks/12-aws-access-containment-follow-up.md` and asked Codex to verify.
- **Applied by:** User/operator
- **Execution location:** external to Codex - AWS CLI with MFA-backed IAM user session
- **Systems affected:** IAM user `MSP-user`, IAM role `sensorglobal-admin`, policy attachment `AdministratorAccess`
- **Summary:** Detached direct `AdministratorAccess` from `MSP-user`, deleted the `MSP-user` console login profile, detached `AdministratorAccess` from the orphaned OneLogin admin role `sensorglobal-admin`, and deleted the `sensorglobal-admin` role.
- **Verification:** Codex ran read-only verification. `iam list-attached-user-policies --user-name MSP-user` returns an empty list. `iam get-login-profile --user-name MSP-user` returns `NoSuchEntity`. `iam get-role --role-name sensorglobal-admin` returns `NoSuchEntity`. `iam list-entities-for-policy --policy-arn arn:aws:iam::aws:policy/AdministratorAccess` now shows no direct users, and only roles `cloudfromation-template-role` and `AWS-QuickSetup-StackSet-Local-ExecutionRole` plus groups `Admin` and `SuperAdmin`. CloudTrail shows `DetachUserPolicy` for `MSP-user` at 2026-05-08 08:22:54 local time, `DeleteLoginProfile` at 08:25:27, `DetachRolePolicy` for `sensorglobal-admin` at 08:29:22, and `DeleteRole` at 08:29:33, all from `peresada1@gmail.com` with `mfaAuthenticated=true`.
- **Notes:** `MSP-user` still exists but currently has no direct admin policy, no login profile, and no access keys. Remaining broad access is documented in `docs/investigations/2026-05-08-aws-admin-access-and-asg-scale-down.md`.

### 2026-05-08 - Appinventiv OneLogin and Backup-user AWS access contained
- **Approval reference:** User asked how to disable Appinventiv/OneLogin AWS access, then confirmed the containment commands were completed and requested verification.
- **Applied by:** User/operator
- **Execution location:** external to Codex - AWS CLI with MFA-backed IAM user session
- **Systems affected:** IAM SAML provider `arn:aws:iam::747293622182:saml-provider/onelogin`, IAM user `Backup-user`, access key `AKIA237RLX6TDZFDABN2`, policy attachment `AdministratorAccess`, IAM group `Admin`
- **Summary:** Set the `Backup-user` access key inactive, deleted the OneLogin/Appinventiv SAML provider used to assume `sensorglobal-admin`, removed `lokendra.saini@appinventiv.com` from IAM group `Admin`, detached `AdministratorAccess` from `Backup-user`, deleted the `Backup-user` access key, and deleted the `Backup-user` IAM user.
- **Verification:** Codex ran read-only verification. `iam list-saml-providers` no longer shows `onelogin`; only `DEV-VPN` remains. `iam get-user --user-name Backup-user` returns `NoSuchEntity`. CloudTrail shows `UpdateAccessKey` setting `AKIA237RLX6TDZFDABN2` inactive at 2026-05-08 08:00:34 local time, `DeleteSAMLProvider` at 08:00:46, `RemoveUserFromGroup` for `lokendra.saini@appinventiv.com` from `Admin` at 08:00:57, `DetachUserPolicy` at 08:01:08, `DeleteAccessKey` at 08:01:18, and `DeleteUser` at 08:01:27, all from `peresada1@gmail.com` with `mfaAuthenticated=true`.
- **Notes:** This was followed by removal of the orphaned `sensorglobal-admin` role and `MSP-user` direct admin access. Remaining AWS entry points and follow-up recommendations are documented in `docs/investigations/2026-05-08-aws-admin-access-and-asg-scale-down.md`.

### 2026-05-08 - Production API and SSO ASG capacity restored
- **Approval reference:** User requested production recovery after Codex identified zero-target ALB `503` impact, then confirmed the recovery commands were applied.
- **Applied by:** User/operator
- **Execution location:** external to Codex - AWS CLI or AWS console with production AWS access
- **Systems affected:** Auto Scaling groups `CodeDeploy_Smoke-API_d-NNRF41DGD` and `CodeDeploy_sso-api-dg_d-QKLSZP50D`, target groups `SmokeAPI` and `sso-api-tg`, public endpoints `api.sensorglobal.com` and `auth.sensorglobal.com`
- **Summary:** Restored production API ASG capacity to `min=2`, `max=5`, `desired=2` and production SSO ASG capacity to `min=1`, `max=2`, `desired=1` after a prior console/API update had set both groups to `min=0`, `max=0`, `desired=0`.
- **Verification:** Codex ran read-only verification. API ASG shows two healthy `InService` instances, SSO ASG shows one healthy `InService` instance, API target group `SmokeAPI` has two healthy targets, SSO target group `sso-api-tg` has one healthy target, SSO discovery returns `HTTP 200`, and `scripts/health-check.sh --env prod --profile sensorsyn-mfa` reports `ALB healthy hosts 2`, `ALB 5xx error rate 0.0%`, `EC2 instances ok/total 4/4`, and no failing infrastructure metrics. Backend API-specific health remained skipped because `HEALTH_API_KEY_PROD` was not present in the local shell.
- **Notes:** CloudTrail identified the preceding scale-to-zero change at 2026-05-07 21:11 Australia/Sydney from an assumed `sensorglobal-admin` session for `chaitanya.sharma@appinventiv.com`. Codex did not run the production recovery mutation.

### 2026-05-07 - Safer Ops production EKS foundation provisioned
- **Approval reference:** User approved prod Terraform edits to provision the Safer Ops EKS cluster and later confirmed the cluster was provisioned and connectable.
- **Applied by:** User
- **Execution location:** external to Codex - Terraform apply with MFA-backed AWS access
- **Systems affected:** EKS, EC2 managed node group, IAM roles, KMS, CloudWatch Logs, and VPC subnet tags in account `747293622182`, region `ap-southeast-2`
- **Summary:** Provisioned EKS cluster `safer-ops-prod` with Kubernetes `1.35`, two `t3.medium` AL2023 managed nodes in private subnets, EKS managed add-ons `vpc-cni`, `kube-proxy`, and `coredns`, KMS secret encryption key `alias/safer-ops-eks`, control-plane log group `/aws/eks/safer-ops-prod/cluster`, EKS cluster role `SaferOpsEksClusterRole`, and node role `SaferOpsEksNodeRole`. Added Kubernetes discovery tags to the existing public and private app subnets for future load balancer integration.
- **Verification:** User confirmed the cluster was provisioned and connectable. Codex ran read-only verification: `kubectl get nodes -o wide` showed two nodes `Ready` on Amazon Linux 2023 with Kubernetes `v1.35.4-eks-40737a8`; `kubectl get pods -A` showed `aws-node`, `coredns`, and `kube-proxy` pods all `Running`.
- **Notes:** Codex prepared and validated the Terraform edits and plans but did not run the remote apply. The cluster API endpoint is `https://9425A24873F732096BDDE6819365937D.gr7.ap-southeast-2.eks.amazonaws.com`.

### 2026-05-07 - Safer Ops EKS IAM OIDC provider registered
- **Approval reference:** User approved prod Terraform edits for the EKS OIDC provider, then confirmed the Terraform apply output with `safer_ops_eks_oidc_provider_arn`.
- **Applied by:** User
- **Execution location:** external to Codex - Terraform apply with MFA-backed AWS access
- **Systems affected:** IAM OIDC provider in account `747293622182`, region `ap-southeast-2`
- **Summary:** Registered the EKS OIDC issuer as IAM OIDC provider `arn:aws:iam::747293622182:oidc-provider/oidc.eks.ap-southeast-2.amazonaws.com/id/9425A24873F732096BDDE6819365937D` with client id `sts.amazonaws.com`, enabling IRSA for future Kubernetes service account roles such as AWS Load Balancer Controller and External DNS.
- **Verification:** User-provided Terraform output confirmed `safer_ops_eks_oidc_provider_arn`, `safer_ops_eks_oidc_issuer`, cluster name, endpoint, node role ARN, and cluster security group ID. Codex verified before apply that the plan was `1 to add, 0 to change, 0 to destroy` and that the only planned AWS resource was `aws_iam_openid_connect_provider.safer_ops_eks`.
- **Notes:** Codex prepared and validated the Terraform edits and plan but did not run the remote apply. The Terraform `tls` provider is used to derive the OIDC thumbprint from the live issuer certificate.

### 2026-05-06 - Safer Ops ECR image publishing bootstrap
- **Approval reference:** User requested AWS CLI bootstrap script for Safer Ops image publishing and confirmed the script completed with image publisher role ARN `arn:aws:iam::747293622182:role/SaferOpsGitHubImagePublisherRole`.
- **Applied by:** User
- **Execution location:** external to Codex - AWS CLI with MFA profile
- **Systems affected:** ECR and IAM in account `747293622182`, region `ap-southeast-2`
- **Summary:** Created ECR repositories `safer-ops-api` and `safer-ops-web`, GitHub Actions OIDC provider `token.actions.githubusercontent.com`, IAM role `SaferOpsGitHubImagePublisherRole`, IAM policy `SaferOpsEcrImagePublisher`, and attached the policy to the role. The role trust policy allows `repo:SaferHomesAu/safer-ops:environment:production` to assume the role through GitHub OIDC.
- **Verification:** Codex ran read-only verification. Both ECR repositories exist; `safer-ops-api` URI is `747293622182.dkr.ecr.ap-southeast-2.amazonaws.com/safer-ops-api`, and `safer-ops-web` URI is `747293622182.dkr.ecr.ap-southeast-2.amazonaws.com/safer-ops-web`. The GitHub OIDC provider exists with `sts.amazonaws.com` as a client id. The publisher role exists with the expected production-environment trust policy and has `SaferOpsEcrImagePublisher` attached. GitHub Actions successfully published API and web images tagged `main` and `9515e143032724562233d21b0c9219771f71d004` to ECR.
- **Notes:** `safer-ops-api` was first created manually in the AWS console and currently has scan-on-push disabled; `safer-ops-web` was created by the script with scan-on-push enabled. Codex did not perform the AWS mutations.

### 2026-05-05 - Safer Ops production SSO client registered
- **Approval reference:** User approved creating `safer_ops_client` in the production SSO Mongo `clients` collection with redirect URI `https://safer-ops.sensorglobal.com/auth/callback`, then confirmed the Atlas insert was completed.
- **Applied by:** User
- **Execution location:** external to Codex - MongoDB Atlas UI
- **Systems affected:** Production SSO MongoDB Atlas collection `sensorproddb.clients`
- **Summary:** Added the Safer Ops OIDC web client `safer_ops_client` for production SSO login. The client allows `authorization_code` and `refresh_token` grants, uses scope `openid email profile offline_access api:read`, and has a registered callback at `https://safer-ops.sensorglobal.com/auth/callback`.
- **Verification:** Codex ran read-only verification against the production SSO MongoDB collection using the existing `sensor-prod-sso` secret. The document exists, `client_secret` is present, and the verified fields match the approved client id, redirect URI, grants, scope, application type, and `__v=0`.
- **Notes:** Codex did not perform the production MongoDB insert. Keep the generated client secret only in deployment secret storage and local secure `.env` files.

### 2026-05-01 - MongoDB custody KMS key and S3 bucket
- **Approval reference:** User requested implementation and executed the AWS commands during the 2026-05-01 MongoDB custody migration walkthrough.
- **Applied by:** User
- **Execution location:** external to Codex - AWS CLI
- **Systems affected:** AWS KMS and S3 in account `747293622182`, region `ap-southeast-2`
- **Summary:** Created customer-managed KMS key `191ab3a4-a352-4d12-b057-610adae73a0c` for MongoDB custody dump encryption, created alias `alias/sensor-prod-mongo-custody`, and configured S3 bucket `sensor-prod-mongo-custody-747293622182` with public access blocked, versioning enabled, and default SSE-KMS encryption using the custody alias.
- **Verification:** User-provided AWS CLI output confirmed KMS key state `Enabled`, public access block all `true`, bucket versioning `Enabled`, and bucket encryption `SSEAlgorithm=aws:kms` with `KMSMasterKeyID=alias/sensor-prod-mongo-custody`.
- **Notes:** No MongoDB dump or restore was performed in this step.

### 2026-05-01 - MongoDB migration runner EC2 instance
- **Approval reference:** User requested implementation and executed the AWS commands during the 2026-05-01 MongoDB custody migration walkthrough.
- **Applied by:** User
- **Execution location:** external to Codex - AWS CLI
- **Systems affected:** EC2, IAM instance profile, and security group in account `747293622182`, region `ap-southeast-2`
- **Summary:** Created IAM role/profile `MongoMigrationRunnerRole` / `MongoMigrationRunnerProfile`, security group `sg-0dd980e523ba78d23`, and temporary migration runner EC2 instance `i-0c179962eb08ed4c6` for SSM-only MongoDB dump/restore work.
- **Verification:** User-provided AWS CLI output confirmed SSM registration for `i-0c179962eb08ed4c6` with `PingStatus=Online`, platform `Amazon Linux`, SSM agent `3.3.4108.0`. Earlier read-only checks confirmed the security group has no ingress and all outbound egress. User fixed a MongoDB repo URL typo, installed `mongodump`/`mongorestore` `100.16.0` and `mongosh` `2.8.2`, then uploaded `dumps/test/custody-test.txt` to the custody bucket with `ServerSideEncryption=aws:kms` and KMS key `191ab3a4-a352-4d12-b057-610adae73a0c`.
- **Notes:** Instance is temporary and should be terminated after dump/restore validation is complete. No production MongoDB dump or restore was performed in this step.

### 2026-05-01 - MongoDB custody dump uploaded to S3
- **Approval reference:** User requested implementation and executed the upload during the 2026-05-01 MongoDB custody migration walkthrough.
- **Applied by:** User
- **Execution location:** external to Codex - AWS CLI from migration runner
- **Systems affected:** S3 bucket `sensor-prod-mongo-custody-747293622182` in account `747293622182`, region `ap-southeast-2`
- **Summary:** Uploaded the completed production `mongodump --archive --gzip` file and its `.sha256` checksum into the custody bucket under `dumps/final/`.
- **Verification:** User confirmed the upload completed in chat. Object-level verification with `aws s3 ls` and `aws s3api head-object` should still be retained if available, especially to confirm SSE-KMS encryption, KMS key, size, version ID, and final object key.
- **Notes:** This is a custody copy only. It does not change production application configuration or production traffic.

### 2026-05-01 - MongoDB custody dump restored to JV Atlas cluster
- **Approval reference:** User requested implementation and executed the restore during the 2026-05-01 MongoDB custody migration walkthrough.
- **Applied by:** User
- **Execution location:** external to Codex - MongoDB Database Tools from migration runner
- **Systems affected:** MongoDB Atlas cluster `sensor-prod` in organization `Sensor Global Pty Ltd`, project `Sensor Production`
- **Summary:** Restored the production `mongodump --archive --gzip` custody dump into the new JV-controlled Atlas cluster. The first restore attempt failed due Atlas write-blocking from target disk exhaustion; after target storage was increased, the restore was rerun successfully.
- **Verification:** User-provided restore output ended with `73138048 document(s) restored successfully. 0 document(s) failed to restore.` The largest restored collection `sensorproddb.tbl_sim_event_logs` completed with `68334311 documents, 0 failures`. Post-restore target validation showed `16` collections, `1` view, `73137634` objects, `100` indexes, and a successful temporary write/drop test.
- **Notes:** Application cutover has not yet been performed. Production secrets and application traffic still require separate controlled cutover and rollback gates.

### 2026-05-01 - Production MongoDB application cutover to JV Atlas
- **Approval reference:** User requested "proceed" after successful restore validation and executed the approved Secrets Manager updates and PM2 restarts during the 2026-05-01 MongoDB custody migration walkthrough.
- **Applied by:** User; Codex ran read-only health and log verification afterward.
- **Execution location:** external to Codex - AWS CLI/SSM and Secrets Manager
- **Systems affected:** Production Secrets Manager secrets `sensor-prod` and `sensor-prod-sso`; production API and SSO PM2 processes; MongoDB Atlas target cluster `sensor-prod` in organization `Sensor Global Pty Ltd`
- **Summary:** Updated `sensor-prod.MONGO_DB_URL` and `sensor-prod-sso.MONGODB_URI` to the new JV-controlled Atlas cluster, then restarted root-owned PM2 processes for the production API and SSO instances so they would reload Secrets Manager values.
- **Verification:** Post-cutover `scripts/health-check.sh --env prod` passed and saved `tmp/baseline-after-mongo-cutover-20260501-133515.json`: ALB healthy hosts `2`, ALB 5xx `0.0%`, backend API `HTTP 200`, Redis connected, stalled hubs `5`, invalid invoices `1`, and SMS sends last 24h `53`. CloudWatch Logs showed recent `mongoose.connection.readyState >>> 1` entries after the cutover.
- **Notes:** `MONGO_DB_ARCHIVE_URL` was intentionally left unchanged for the initial cutover, but post-restart logs showed `MONGO_DB_ARCHIVED` connection errors against the old Atlas Online Archive endpoint. Treat archive recreation/retargeting as a follow-up before relying on historical archived data. Rotate/delete any temporary Atlas migration user whose URI appeared in terminal output.

### 2026-05-01 - Production MongoDB URI encoding fix
- **Approval reference:** User reported fresh incognito login failure after cutover, then updated the Atlas DB user passwords/URIs and restarted the affected services.
- **Applied by:** User; Codex ran read-only verification afterward.
- **Execution location:** external to Codex - AWS CLI/SSM and Secrets Manager
- **Systems affected:** Production Secrets Manager secrets `sensor-prod` and `sensor-prod-sso`; production API and SSO PM2 processes
- **Summary:** Corrected malformed MongoDB Atlas connection strings that contained unescaped characters in URI userinfo, then restarted root-owned PM2 processes for the production API and SSO instances.
- **Verification:** Restart command `e33de6af-b10a-4b19-b152-95a3a82d9856` succeeded on the three API instances and the SSO instance. Redacted URI diagnostics showed both `sensor-prod.MONGO_DB_URL` and `sensor-prod-sso.MONGODB_URI` point to `sensor-prod.yujwyg.mongodb.net` with no unsafe raw characters in userinfo. CloudWatch Logs showed fresh `Connected to MONGO_DB in bootstrap` events and no fresh `MongoParseError` in the checked window. `scripts/health-check.sh --env prod` saved `tmp/baseline-after-mongo-uri-fix-20260501-141353.json`; infrastructure checks passed, while backend API-specific checks were skipped because `HEALTH_API_KEY` was not present in the local shell.
- **Notes:** Confirm fresh browser login/logout manually. The Atlas Online Archive follow-up remains open because `MONGO_DB_ARCHIVE_URL` was not retargeted in this fix.

### 2026-05-01 - Production MongoDB database path fix
- **Approval reference:** User reported `client authentication failed` on fresh incognito login after the URI encoding fix; Codex diagnosed that both Mongo URIs were missing the `/sensorproddb` database path.
- **Applied by:** User; Codex ran read-only verification afterward.
- **Execution location:** external to Codex - AWS CLI/SSM and Secrets Manager
- **Systems affected:** Production Secrets Manager secrets `sensor-prod` and `sensor-prod-sso`; production API and SSO PM2 processes
- **Summary:** Updated `sensor-prod.MONGO_DB_URL` and `sensor-prod-sso.MONGODB_URI` so both connection strings target `/sensorproddb` on the new JV Atlas host, then restarted the three production API PM2 processes and the production SSO PM2 process.
- **Verification:** Restart command `c824f9bc-ba25-461c-82f0-3c39fee15196` succeeded on the three API instances and the SSO instance. Redacted URI diagnostics showed both `MONGO_DB_URL` and `MONGODB_URI` target host `sensor-prod.yujwyg.mongodb.net` with path `/sensorproddb`. CloudWatch Logs showed fresh `Connected to MONGO_DB in bootstrap` events, no fresh `MongoParseError`, no SSO `client authentication failed` entries, and no fresh SSO error entries in the checked window.
- **Notes:** Confirm fresh browser login/logout manually. The Atlas Online Archive follow-up remains open because `MONGO_DB_ARCHIVE_URL` was not retargeted in this fix.

### 2026-05-01 - Production SSO basemodels index compatibility fix
- **Approval reference:** User reported `400 VALIDATION_ERROR` / `oops! something went wrong` on login after the database path fix; Codex diagnosed an extra unique `payload.grantId_1` index on the restored JV Atlas `sensorproddb.basemodels` collection that was absent from the old Atlas cluster.
- **Applied by:** User; Codex ran read-only diagnosis afterward.
- **Execution location:** external to Codex - MongoDB Atlas / `mongosh`
- **Systems affected:** MongoDB Atlas cluster `sensor-prod` in organization `Sensor Global Pty Ltd`, collection `sensorproddb.basemodels`; production SSO login flow
- **Summary:** Dropped the extra `payload.grantId_1` unique index from `sensorproddb.basemodels`. The index was incompatible with the SSO `oidc-provider` storage pattern because successful login writes multiple token/session records that can share a grant id.
- **Verification:** Direct SSO password grant for the real test account previously returned `500 server_error` before the index fix, while bad-password/fake-user paths returned expected credential errors. After the index was dropped, the user confirmed browser login succeeded.
- **Notes:** No service restart was expected for this index-only fix. Continue manual login/logout checks and monitor SSO/API logs during the soak window.

## Entry Template

### YYYY-MM-DD - Short change title
- Approval reference:
- Applied by:
- Execution location: external to Codex | local repo only
- Systems affected:
- Summary:
- Verification:
- Notes:

---

### 2026-04-28 — AWS account detached from Ingram Micro parent organization
- **Approval reference:** User confirmed in chat: "ok, got approval. created additional account. did detach"
- **Applied by:** User / JV administrator
- **Execution location:** external to Codex — AWS Organizations member-account detach executed outside Codex
- **Systems affected:** AWS account `747293622182`; AWS Organizations membership and billing/governance boundary
- **Summary:** Account `747293622182` was detached from the Ingram Micro parent AWS Organization and now operates as a standalone AWS account under JV control. Workloads remained in the same AWS account; this was not a workload migration.
- **Verification:** `aws organizations describe-organization` now returns `AWSOrganizationsNotInUseException: Your account is not a member of an organization.` `aws sts get-caller-identity` still confirms account `747293622182`. Post-detach health check saved to `tmp/baseline-after-account-detach-20260428-144824.json` with ALB healthy hosts `3`, ALB 5xx `0.0%`, RDS CPU `5.5%`, RDS free storage `90.0 GB`, RDS connections `15`, EC2 status `7/7`, and SMS sends last 24h `107`. Route 53 Domains list still shows 14 domains visible with AutoRenew enabled.
- **Notes:** Root user ownership remains unrecovered and should be handled as a follow-up risk item. User later confirmed a new payment method was added in the AWS console on 2026-04-28. Billing alternate contact is still missing; operations and security alternate contacts are configured. Support API still reports `SubscriptionRequiredException`, so support plan verification remains pending in the AWS console if Business/Premium support is desired.

---

### 2026-04-11 — CloudWatch log retention set on non-prod environments
- **Approval reference:** Cost optimisation session, user approved retention automation
- **Applied by:** Copilot (scripts/set-log-retention.sh)
- **Execution location:** local repo — script ran against AWS API
- **Systems affected:** All CloudWatch log groups in dev, qa, sandbox
- **Summary:** Set 30-day retention on all log groups that had no retention policy (previously retained forever). Logged group names + previous settings before changing.
- **Verification:** `aws logs describe-log-groups` confirmed retention set; no application errors observed
- **Notes:** Script supports `--dry-run`; prod log groups were intentionally skipped

---

### 2026-04-12 — Terraform IaC baseline: all 5 environments imported
- **Approval reference:** User approved iterative Terraformer import across session turns
- **Applied by:** Copilot (terraform import, terraform state mv)
- **Execution location:** local repo — `code/infra/terraform/environments/{account,prod,dev,qa,sandbox}/`
- **Systems affected:** Terraform state only — no AWS resources were modified
- **Summary:** All existing infrastructure (VPCs, subnets, SGs, EC2, RDS, ALB, ElastiCache, EIPs, IAM roles, S3 buckets, Route53, ACM certs, CloudWatch) imported into Terraform state across all 5 env modules. State stored in S3 backend `sensorsyn-terraform-state-747293622182`. All 5 environments verified at `No changes` after import.
- **Verification:** `terraform plan` across all 5 envs shows `No changes. Your infrastructure matches the configuration.`
- **Notes:** Import runbook at `code/infra/terraform/docs/import-runbook.md`; account-level resources (Route53, shared ACM cert, IAM SSM role) isolated in `account/` module

---

### 2026-04-14 — Terraform workflow conventions added to copilot-instructions.md
- **Approval reference:** User approved "apply" of session-learned improvements
- **Applied by:** Copilot (edit to .github/copilot-instructions.md)
- **Execution location:** local repo only
- **Systems affected:** `.github/copilot-instructions.md`
- **Summary:** Added "Terraform workflow conventions" section with 4 rules learned from session: (1) execution posture — run plans/applies directly; (2) import safety — check all module states first, account-level resources only in `account/`; (3) destroy guard — any destroy in plan = stop and diagnose; (4) ALB cert rotation — purge expired SNI entries after attaching new cert.
- **Verification:** Section present in file; no AWS changes
- **Notes:** Rules prevent repeat of prod plan destroy incident and duplicate ACM/IAM ownership issue discovered during import

---

### 2026-04-16 — Locals extraction: hardcoded AWS IDs consolidated into locals.tf per environment
- **Approval reference:** User approved locals extraction as part of migration baseline work
- **Applied by:** Copilot (scripts/locals-extract.py)
- **Execution location:** local repo — script ran `terraform show -json` to pull state then rewrote .tf files
- **Systems affected:** `code/infra/terraform/environments/{dev,qa,sandbox,prod}/` — .tf files only, no AWS changes
- **Summary:** All hardcoded AWS resource IDs (vpc-xxx, sg-xxx, subnet-xxx, ami-xxx, eipalloc-xxx, eni-xxx, igw-xxx, nat-xxx, rtb-xxx, instance IDs) extracted from inline .tf files into a `locals.tf` per environment. Inline references replaced with `local.<name>` variables. ID counts: dev 48, qa 39, sandbox 49, prod 61.
- **Verification:** `terraform plan` on all 4 envs (account env has no IDs) shows `No changes` post-extraction.
- **Notes:** For migration to a new account, only `locals.tf` per env needs updating — all resource body code is now ID-free. `scripts/locals-extract.py` can be re-run to refresh if new IDs are added.

---

### 2026-04-16 — CloudWatch alarms and Secrets Manager metadata imported into Terraform
- **Approval reference:** User approved automated import of missing alarms/secrets and asked to capture UAT
- **Applied by:** Copilot (`scripts/generate-missing-imports.py`, `terraform import`)
- **Execution location:** local repo — `code/infra/terraform/environments/{dev,qa,sandbox,prod}/`
- **Systems affected:** Terraform state + generated Terraform files only; no AWS resources were modified
- **Summary:** Added generated Terraform config + import blocks for env-scoped CloudWatch alarms and Secrets Manager secrets, then imported them directly into state. Imported counts: dev `42 alarms + 2 secrets`, qa `48 + 2`, sandbox `25 + 2`, prod `84 + 2`.
- **Verification:** `terraform plan` shows `No changes` in dev, qa, and sandbox after import. Prod alarm/secret import completed, but the final full prod plan now shows one unrelated ASG size drift on `aws_autoscaling_group.prod_api_server` (not caused by the alarm/secret imports).
- **Notes:** `scripts/generate-missing-imports.py` classifies UAT/account/local resources but only writes to existing env dirs by default. UAT inventory was captured separately in `docs/investigations/2026-04-16-uat-environment-discovery.md`.

---

### 2026-04-17 — UAT environment scaffolded + stopped variable wired into non-prod envs
- **Approval reference:** User approved UAT keep-and-scaffold + stopped variable for dev/qa/sandbox
- **Applied by:** Copilot (file edits in local repo only)
- **Execution location:** local repo — `code/infra/terraform/environments/{dev,qa,sandbox,uat}/`
- **Systems affected:** Terraform config only — no AWS resources were modified
- **Summary:**
  1. Added `var.stopped` (bool, default false) to `variables.tf` for dev, qa, sandbox, and uat.
  2. Wired `var.stopped` into `desired_capacity` and `min_size` on all 6 non-prod ASGs (dev_sso, dev_api, qa_sso, qa_api, sandbox_sso_asg, sandbox_api_asg).
  3. Created `stopped.tfvars` per env with usage instructions (including RDS/ElastiCache note and `stop-nonprod.sh` cross-reference).
  4. Scaffolded `environments/uat/` with `backend.tf`, `main.tf`, `variables.tf`, `locals.tf` (ID placeholders from discovery doc), and `stopped.tfvars`. Backend initialized and pointing to `nonprod/uat/terraform.tfstate`.
- **Verification:**
  - `terraform validate` passes for dev, qa, sandbox.
  - `terraform plan` (stopped=false default): `No changes` for dev, qa, sandbox, uat.
  - `terraform plan -var-file=stopped.tfvars` for dev: shows `desired_capacity 2→0, min_size 2→0` on both ASGs — `Plan: 0 to add, 2 to change, 0 to destroy`. No destroy operations.
- **Notes:** RDS/ElastiCache stop requires `scripts/stop-nonprod.sh` — Terraform has no native "stop" for these services. UAT `locals.tf` has placeholder comments for IDs pending `terraform import` of UAT resources.

---

### 2026-04-17 — UAT Terraform import completed (base infra + alarms + secrets)
- **Approval reference:** User approved UAT resource import (continuation of IaC baseline work)
- **Applied by:** Copilot (`terraform import`, `scripts/generate-missing-imports.py`)
- **Execution location:** local repo — `code/infra/terraform/environments/uat/`
- **Systems affected:** Terraform state only — no AWS resources were modified
- **Summary:**
  1. Wrote all UAT HCL files from verified AWS inventory: `networking.tf` (VPC, IGW, NAT, 6 subnets, 3 route tables, 16 SGs), `compute.tf` (3 EIPs, 4 EC2, 2 LTs, 2 ASGs), `data.tf` (RDS, ElastiCache, subnet groups), `loadbalancer.tf` (2 ALBs, 2 TGs, 4 listeners), and `locals.tf` (central ID registry).
  2. Imported all ~63 base resources across 7 phases. Fixed SG `description` field to prevent forced replacement; fixed route table associations import format (must use `subnet_id/route_table_id` not `rtbassoc-xxx`).
  3. Reconciled HCL to match live AWS state: fixed ALB `idle_timeout`, `access_logs` bucket names, TG `healthy_threshold`/`interval`/`matcher`, RDS `backup_window`/`maintenance_window`/`parameter_group_name`, ElastiCache `maintenance_window`/`snapshot_window`, ASG `protect_from_scale_in`, VPC peering routes (`192.168.248.0/21` via `pcx-0378fb73738ca8785`), DB subnet group description.
  4. Ran `scripts/generate-missing-imports.py --env uat --write --import` to import 20 CloudWatch alarms and 2 Secrets Manager secrets.
  5. Removed `uat` from `UNSUPPORTED_OUTPUT_ENVS` in the script now that the env dir exists.
- **Verification:** `terraform plan` shows `Plan: 0 to add, 46 to change, 0 to destroy`. The 46 in-place changes are all safe: `default_tags` additions (Environment/ManagedBy/Repo), default attribute population, Name tag cleanups, and one legitimate port correction (433→443 typo on the smoke-uat-api HTTP redirect listener).
- **Notes:** Total UAT state: 85 resources (63 base + 20 alarms + 2 secrets). Cost-optim TODOs remain in `data.tf`: UAT RDS multi_az=true→false and gp2→gp3.

---

### 2026-04-29 — Terraform post-decommission drift reconciled for prod, sandbox, dev, and QA
- **Approval reference:** User requested "prod autoscaling, then sandbox and last dev, qa" and approved proceeding in-session
- **Applied by:** Codex
- **Execution location:** local repo — `code/infra/terraform/environments/{prod,sandbox,dev,qa}/`
- **Systems affected:** Terraform config only — no AWS resources were modified and no Terraform apply/state mutation was run
- **Summary:**
  1. Reconciled prod API ASG config to match live AWS: `min_size=3`, `max_size=3`, warm pool `Running`.
  2. Reconciled sandbox config to current partial-decommissioned state: removed deleted app-tier ASGs from active `.tf`, updated launch template AMIs/default versions to live values, and kept live `365` day retention for `smoke-api-sandbox-pm2-out-log`.
  3. Removed stale dev/QA Secrets Manager import/resource config for deleted secrets.
  4. Moved decommissioned dev/QA generated resource files to `.tf.decommissioned` so Terraform will not recreate those retired stacks while preserving the old imported config for reference.
- **Verification:**
  - `terraform validate` passes for prod, sandbox, dev, and QA.
  - `terraform plan -lock=false -input=false -no-color -detailed-exitcode` shows `No changes. Your infrastructure matches the configuration.` for prod, sandbox, dev, and QA.
- **Notes:** UAT was not included in this cleanup pass and may still need a separate decommissioned-state reconciliation. Prod API ASG config was later updated again on 2026-04-29 to prepare the approved autoscaling change.

---

### 2026-04-29 — Prod API autoscaling applied
- **Approval reference:** User requested "proceed to autoscaling of prod" and then confirmed "applied"
- **Applied by:** Applied externally by user/operator; Codex prepared Terraform config and verified afterward
- **Execution location:** local repo — `code/infra/terraform/environments/prod/generated_prod.tf`
- **Systems affected:** `aws_autoscaling_group.prod_api_server` (`CodeDeploy_Smoke-API_d-NNRF41DGD`) and its warm pool
- **Summary:** Updated prod API ASG autoscaling bounds from fixed `min=max=3` to `min_size=2`, `max_size=5`, with warm pool `pool_state="Stopped"`. `desired_capacity` remains ignored in Terraform so the target tracking policy can own runtime capacity.
- **Verification:** `aws autoscaling describe-auto-scaling-groups` shows live `Desired=3`, `Min=2`, `Max=5` with three healthy InService instances. `aws autoscaling describe-warm-pool` shows warm pool config `PoolState=Stopped`, `MinSize=1`, `InstanceReusePolicy.ReuseOnScaleIn=true`. `terraform plan -lock=false -input=false -no-color -detailed-exitcode` for prod shows `No changes. Your infrastructure matches the configuration.`
- **Notes:** Codex did not run `terraform apply`; the remote change was applied externally and verified from the user's confirmation plus read-only AWS/Terraform checks.

---

### 2026-04-29 — OPT-001 sandbox Terraform decommission preparation applied
- **Approval reference:** User approved OPT-001 with "proceed", asked whether it could be done via Terraform, then confirmed "applied" after the stage-1 Terraform prepare diff was provided.
- **Applied by:** Applied externally by user/operator; Codex prepared Terraform config and verified afterward.
- **Execution location:** local repo — `code/infra/terraform/environments/sandbox/{data.tf,loadbalancer.tf,variables.tf}`
- **Systems affected:** `aws_db_instance.sensor_sandbox`, `aws_elasticache_replication_group.smokealarmsandbox`, `aws_lb.sandbox_api_lb`, `aws_lb.sandbox_sso_lb`
- **Summary:** Prepared sandbox fixed-cost resources for later Terraform decommission by disabling RDS and ALB deletion protection, setting RDS final snapshot behavior, preserving RDS automated backups, and setting final snapshot identifiers for RDS and Redis.
- **Verification:** `terraform plan -lock=false -input=false -no-color -detailed-exitcode` for sandbox shows `No changes. Your infrastructure matches the configuration.` `aws rds describe-db-instances` shows `sensor-sandbox` `DeletionProtection=false`. `aws elbv2 describe-load-balancer-attributes` shows both `Smoke-Sandbox-Api-LB` and `sandbox-sso-api` `deletion_protection.enabled=false`.
- **Notes:** Codex did not run `terraform apply`; the remote change was applied externally and verified from the user's confirmation plus read-only AWS/Terraform checks. This is stage 1 only; stage 2 will remove/gate the selected sandbox resources and produce the actual destroy plan.

---

### 2026-04-29 — OPT-001 sandbox EC2 decommission preparation applied
- **Approval reference:** User confirmed "applied" after the stage-2A Terraform prepare diff was provided.
- **Applied by:** Applied externally by user/operator; Codex prepared Terraform config and verified afterward.
- **Execution location:** local repo — `code/infra/terraform/environments/sandbox/compute.tf`
- **Systems affected:** `aws_instance.cron_sandbox`, `aws_instance.kafka_mongo_sandbox`, `aws_instance.emqx_sandbox`, `aws_instance.odoo_sandbox`
- **Summary:** Prepared stopped sandbox EC2 instances for later Terraform decommission by disabling API termination protection on Odoo/Kafka/EMQX and enabling root-volume delete-on-termination on cron, Kafka, and Odoo.
- **Verification:** `terraform plan -lock=false -input=false -no-color -detailed-exitcode` for sandbox shows `No changes. Your infrastructure matches the configuration.` `aws ec2 describe-instance-attribute` shows `DisableApiTermination=false` for `i-0b71daa2680a16a5d`, `i-07af1c06915741068`, and `i-0074106adaa006f2e`.
- **Notes:** Codex did not run `terraform apply`; the remote change was applied externally and verified from the user's confirmation plus read-only AWS/Terraform checks. This is stage 2A only; stage 2B will remove/gate the selected sandbox resources and produce the actual destroy plan.

---

### 2026-04-29 — OPT-001 sandbox fixed-cost decommission applied
- **Approval reference:** User requested "lets proceed with 2B", applied the Terraform stage externally, reported partial apply errors, then confirmed "this is done" after applying the finish plan.
- **Applied by:** Applied externally by user/operator; Codex prepared Terraform config/plans and verified afterward.
- **Execution location:** external Terraform apply from `code/infra/terraform/environments/sandbox`
- **Systems affected:** `sensor-sandbox` RDS, `smokealarmsandbox` ElastiCache, sandbox NAT Gateway `nat-0f00d13d55f11c828`, NAT EIP `eipalloc-0121eef619a926133`, sandbox API/SSO ALBs/listeners/target groups, stopped sandbox EC2 instances `i-0074106adaa006f2e`, `i-0b71daa2680a16a5d`, `i-008e2acd8997ff315`, `i-07af1c06915741068`, `i-0c84b9424bc1ce4dd`, and their Terraform-managed EIPs.
- **Summary:** Removed the selected sandbox fixed-cost resources from active Terraform and decommissioned them externally. The first apply partially completed and reported stale `ListenerNotFound` errors plus a NAT EIP association error; the follow-up finish plan removed the remaining unassociated NAT EIP.
- **Verification:** Final sandbox `terraform plan -lock=false -input=false -no-color -detailed-exitcode` shows `No changes. Your infrastructure matches the configuration.` Read-only AWS checks show `sensor-sandbox` and `smokealarmsandbox` are not found, NAT Gateway `nat-0f00d13d55f11c828` is `deleted`, NAT EIP `eipalloc-0121eef619a926133` is not found, the five stopped sandbox EC2 instances are terminated, the five sandbox instance EIPs are released, and the sandbox API/SSO ALBs are not found.
- **Notes:** Codex did not run `terraform apply` or destructive AWS APIs. Route53 sandbox records are outside this Terraform stage and may still need cleanup if they point at deleted ALBs or released EIPs. Stage 2B estimated gross recurring saving is about `USD 445/month` / `AUD 630/month` before taxes, discounts, partial-month timing, and residual snapshot or backup storage.

---

### 2026-05-06 — Production RDS storage gp3 optimisation applied externally
- **Approval reference:** User requested the RDS cost optimisation, ran the Terraform apply externally, and reported "applied".
- **Applied by:** Applied externally by user/operator; Codex prepared Terraform config/plans and verified read-only afterward.
- **Execution location:** external Terraform apply from `code/infra/terraform/environments/prod`
- **Systems affected:** `aws_db_instance.odoo_production` (`odoo-production`) and `aws_db_instance.sensor_prod` (`sensor-prod`)
- **Summary:** Production RDS storage-class optimisation moved `odoo-production` from `io1` to `gp3` and `sensor-prod` from `gp2` to `gp3` without changing instance class or Multi-AZ posture. Both Terraform resources omit explicit gp3 IOPS/throughput for sub-400 GiB RDS storage so AWS uses the included baseline.
- **Verification:** Read-only `aws rds describe-db-instances` after the external immediate apply showed both DB instances on current `StorageType=gp3`, `Iops=3000`, `StorageThroughput=125`, and no pending modified values. Both were in `storage-optimization`; RDS events showed each storage modification finished applying. `sensor-prod` CloudWatch `DatabaseConnections` stayed flat at 2 through the checked window.
- **Notes:** Codex did not run `terraform apply` or any mutating AWS API. Terraform prod defaults and optimisation varfiles were updated afterward so future plans do not try to revert `sensor-prod` to `gp2` or set invalid explicit gp3 throughput/IOPS for sub-400 GiB RDS instances.

---

### 2026-05-15 — Safer Ops production DNS alias applied externally
- **Approval reference:** User approved/proceeded with the Safer Ops production DNS Terraform step and later confirmed "deployed, looks like resolving and traffic is routing".
- **Applied by:** Applied externally by user/operator; Codex prepared the Terraform Route53 record and verified the record afterward.
- **Execution location:** external Terraform apply from `code/infra/terraform/environments/prod`
- **Systems affected:** Route53 public hosted zone `sensorglobal.com` (`Z03520551S3PY3J02L5ZN`) and Safer Ops production ALB `safer-ops-prod-854086076.ap-southeast-2.elb.amazonaws.com`
- **Summary:** Added `safer-ops.sensorglobal.com` as an `A` alias record to the Safer Ops production ALB.
- **Verification:** Read-only Route53 lookup showed `safer-ops.sensorglobal.com.` has an `A` alias target of `safer-ops-prod-854086076.ap-southeast-2.elb.amazonaws.com.` with hosted zone ID `Z1GM3OXH4ZPM65`.
- **Notes:** Codex did not run `terraform apply`, mutating AWS APIs, or live production HTTP/Kubernetes diagnostics. HTTPS traffic routing was confirmed by the user externally.

---

### 2026-06-23 — Demo prep: agency/agent account 59120 phone typo corrected (SMS whitelist match)
- **Approval reference:** User instructed "lets update agency number to whitelisted, its 8 there (same as tenant)" during demo-readiness prep for properties 23854/23855.
- **Applied by:** Claude, via tightly-scoped transactional write script (`/tmp/agency-phone-fix.js`) dispatched over SSM to prod SmokeAPI node `i-09e470062a5ff6ba1`; mirrors the approved `scripts/diag/sensor-admin-flag-reset.js` pattern (FOR UPDATE snapshot, exact-row guard, before/after).
- **Execution location:** prod MySQL (`sensor-prod`), `tbl_admins`, on-instance cred fetch (never printed).
- **Systems affected:** `tbl_admins` row `id=59120` (Kristyn Heywood, agency/agent account, `kristyn.heywood@gmail.com`).
- **Summary:** Corrected typo'd phone `401596481` → `401586481` (phoneCode 61 unchanged) so agent-targeted SMS clears `tbl_sms_allowed_list` (whitelisted entry id 3 = +61 401586481, same number as tenant account 59280). Needed for the demo's 15-min tamper agent SMS and any other agent SMS to actually deliver.
- **Verification:** Script reported BEFORE phone `401596481`, AFTER `401586481`, `affected_rows=1`. Guard asserted exactly one row with the old value before updating.
- **Undo:** `UPDATE tbl_admins SET phone='401596481' WHERE id=59120`
- **Notes:** Account 59120 is the test agency (agency=agent=59120 on both demo properties). `alertConfiguration.tampered_for_15_minutes.agent` was already `sms:true,email:true` on both properties, so no config change was needed.

---

### 2026-06-23 — Demo: pre-test "warn" notification sent to tenant on 23854 (net-zero)
- **Approval reference:** User instruction "Send test message to Kristyn" during the live install demo (13A Peters / property 23854).
- **Applied by:** Claude. Set/restore `alarmTestDate` via scoped transactional write (`/tmp/testdate-write.js`, FOR UPDATE + exact-row guard) on prod API node `i-09e470062a5ff6ba1`; cron triggered from the whitelisted cron box `i-0392d1fca678f1a21` (Basic-auth token never left the box).
- **Execution location:** prod MySQL `tbl_properties` + cron box HTTP GET `/admins/properties/test-alarm-notify-cron`.
- **Systems affected:** property 23854 `alarmTestDate` (temporarily) → triggered email S035 + SMS S056 to active-lease tenant 59280 (Kristyn, `@saferhomesau`, +61 401586481).
- **Summary:** Gates verified first (property ACTIVE(1), job 6458 COMPLETED(5), recipient whitelisted). Set `alarmTestDate` = now+24h (2026-06-24T02:21:00Z), fired notify-cron once (HTTP 200), then restored `alarmTestDate` to original `2027-06-23T02:00:00Z`.
- **Verification:** `tbl_logs` (Mongo) for propertyId 23854 shows 2 rows @ 02:22:00Z — Email "Sensor Global - Scheduled Notification of Smoke Alarm Test" + Sms "Alarm Test". `alarmTestDate` AFTER = 2027-06-23T02:00:00Z (restored).
- **Undo:** none needed — net-zero (set then restored). The dispatched comm is intentional.
- **Note:** First trigger attempt failed (exit 127) because the `sed` left the cron schedule `*/30 * * * *` before `curl`; fixed by extracting `grep -o 'curl .*'`. Re-fired within the same UTC half-hour bucket.

---

### 2026-06-30 — Created append-only time-series collection `tbl_device_events`
- **Approval reference:** User "its a go for new collection" — Phase 1 of the forensic device-event journal (design: `docs/investigations/2026-06-30-device-events-journal-design.md`).
- **Applied by:** Claude, via idempotent provisioning runner `code/sensor-alarm-backend/db/mongo-migrations/2026-06-30-create-device-events.js` dispatched over SSM to prod API node `i-0a80db6a2df9c3edf` (Mongo URL fetched from the `sensor-prod` secret on-box, never printed).
- **Execution location:** prod MongoDB Atlas `sensorproddb` (live), v8.0.26.
- **Systems affected:** new collection `sensorproddb.tbl_device_events` (+ backing `system.buckets.tbl_device_events`). No existing data touched.
- **Summary:** Created a time-series collection (`timeField: ts`, `metaField: meta`, `granularity: minutes`) with `expireAfterSeconds=31536000` (365-day TTL) and 6 indexes: auto `(meta,ts)` + `meta.hubSerial_ts`, `propertyId_ts`, `childSerial_ts`, `correlationId`, `testId`. Empty; will receive ground-truth MQTT frames from the forthcoming `mqtt-forensic-tap`.
- **Verification:** Runner reported `isTimeSeries:true`, `nindexes:6`, `createCollection created:true`. Independent read-only `$collStats` confirmed the time-series backing ns `sensorproddb.system.buckets.tbl_device_events` and all 6 named indexes; collection empty.
- **Undo:** `db.tbl_device_events.drop()`
- **Notes:** First Mongo migration in the repo (`db/mongo-migrations/`). Idempotent + version-gated (measurement-field indexes require Mongo ≥6.3; cluster is 8.0.26, so all created).

---

### 2026-06-30 — Created `SaferHomesForensicTapRole` (IRSA) + `mqtt-forensic-tap` ECR repo
- **Approval reference:** User named-resource ack "go — SaferHomesForensicTapRole + mqtt-forensic-tap ECR" — deployment prerequisites for the forensic MQTT tap (runbook `docs/runbooks/14-deploy-mqtt-forensic-tap.md`).
- **Applied by:** Claude, `terraform apply` of a **targeted, saved plan** from `code/infra/terraform/environments/prod` under `AWS_PROFILE=sensorsyn-mfa` (file `saferhomes_forensic_tap.tf`, renamed from `.tf.proposed`).
- **Execution location:** AWS account `747293622182`, prod Terraform (S3 backend `sensorsyn-terraform-state-747293622182`).
- **Systems affected:** NEW `aws_iam_role.SaferHomesForensicTapRole` + `SaferHomesForensicTapSecretsPolicy` + attachment; NEW `aws_ecr_repository.mqtt-forensic-tap` + lifecycle policy. `-target` scoped to exactly these 5; plan showed **5 to add, 0 to change, 0 to destroy** (no unrelated prod drift touched).
- **Summary:** IRSA role trust-bound to `system:serviceaccount:saferhomes-workers:mqtt-forensic-tap` via the cluster OIDC provider; permission = `secretsmanager:GetSecretValue`/`DescribeSecret` on the `sensor-prod` secret ARN ONLY. ECR repo: scan-on-push, expire-untagged-after-14d. Lets External Secrets read `sensor-prod` for the tap; does NOT widen the safer-ops IRSA.
- **Verification:** `Apply complete! Resources: 5 added, 0 changed, 0 destroyed`. Independent read-only: `aws iam get-role` (role exists, created `2026-06-30T23:14:52Z`) with `SaferHomesForensicTapSecretsPolicy` attached; `aws ecr describe-repositories` (`mqtt-forensic-tap`, `scanOnPush:true`). Outputs: role `arn:aws:iam::747293622182:role/SaferHomesForensicTapRole`, ECR `747293622182.dkr.ecr.ap-southeast-2.amazonaws.com/mqtt-forensic-tap`.
- **Undo:** `terraform destroy -target` the 5 resources (or `aws iam delete-role`/detach + `aws ecr delete-repository`), and `git mv saferhomes_forensic_tap.tf` back to `.tf.proposed`.
- **Notes:** Manifest `<ACCOUNT_ID>` placeholders filled with `747293622182`. Image build/push + k8s deploy are the remaining follow-ups (runbook steps 2–5).

---

### 2026-07-01 — Deployed `mqtt-forensic-tap` to safer-ops-prod EKS; live ground-truth capture
- **Approval reference:** User "proceed, its a go" for `kubectl apply` of the tap manifests (runbook 14, deploy step).
- **Applied by:** Claude. Cross-built `linux/amd64` image via `docker buildx`, pushed `…/mqtt-forensic-tap:0.1.0` to ECR; `kubectl apply -k code/mqtt-forensic-tap/deploy/k8s` to cluster `safer-ops-prod`.
- **Execution location:** EKS `safer-ops-prod` (account `747293622182`), namespace `saferhomes-workers`.
- **Systems affected:** NEW namespace `saferhomes-workers` + ServiceAccount (IRSA) + SecretStore (v1) + ExternalSecret + headless Service + StatefulSet (1 replica) `mqtt-forensic-tap`. New consumer on prod EMQX (**plain** QoS-2 subscription to `sg/sas/{resp,cmd}/#` under its own clientId — NOT the `sensorGroup` shared sub, so it does not steal frames). Append-only writes to `sensorproddb.tbl_device_events`.
- **Summary:** Forensic tap is live: External Secrets syncs `sensor-prod` creds into the namespace via `SaferHomesForensicTapRole` (IRSA); the pod connects to Mongo + EMQX and captures ground-truth frames. Two rollout fixes: (a) the cluster's External-Secrets CRDs serve only `v1` (not `v1beta1`) → bumped `apiVersion`; (b) removed `spec.provider.aws.role` from the SecretStore — it caused a redundant second `sts:AssumeRole` the role's web-identity-only trust policy rejected (`serviceAccountRef` + the SA's role annotation is the single-hop path).
- **Verification:** pod `1/1 Running`; logs `mongo connected` + `subscribed … qos 2`; ExternalSecret `SecretSynced ready=True` (6 keys); `tbl_device_events` count **524 and climbing** within ~3 min; distribution command/QoS0 = 351, report/QoS2 = 174; events VERIFY/BATTERY/ALARMS/TEST/POWER/LOW_BATTERY; sample doc well-formed.
- **Undo:** `kubectl delete -k code/mqtt-forensic-tap/deploy/k8s` (removes the namespace + workload; `tbl_device_events` + IRSA/ECR remain).
- **Notes:** EARLY FINDING for task #9 — device reports arrive at **QoS 2**, backend commands at QoS 0; the backend's QoS-0 subscription downgrades the entire device-report stream. Wire volume is materially higher than `tbl_alarm_logs`' logged rate; quantify the drop as the QoS-0 investigation.

---

### 2026-07-04 — Added dev workstation to the "Seven Days Backup" DLM policy
- **Approval reference:** User named-resource ack "its a go for dlm" (proposal: cover the dev workstation volume with the existing snapshot policy; runbook `docs/runbooks/15-workstation-recovery.md`, open gap 1).
- **Applied by:** Claude, boto3 `dlm.update_lifecycle_policy` under `AWS_PROFILE=sensorsyn-mfa`.
- **Execution location:** AWS account `747293622182`, ap-southeast-2, DLM policy `policy-0441001e0e7116959` ("Seven Days Backup").
- **Systems affected:** DLM policy TargetTags only — appended `{'Name': 'yevgen-devbox'}` (matches instance `i-0a82d062d3c2976ee`; root volume `vol-02e6411852fc189d2`, 150 GiB gp3, encrypted). No schedule, retention, or existing target changes.
- **Summary:** The dev workstation (Claude workspace: memory store, docs, ops-and-extracts undo files, scripts, toolchain) previously had ZERO snapshots. It now gets the policy's daily 09:00 UTC snapshot with 7-day age-based retention (bounded — no sprawl). Policy `ResourceTypes=['INSTANCE']` with `ExcludeBootVolume:false`, so the root volume is covered via the instance `Name` tag.
- **Verification:** Post-update re-read: state ENABLED, target count 14→15, devbox entry present, all 14 prior targets unchanged, schedule unchanged (`09:00`, retain 7 days). First snapshot expected at the next 09:00 UTC run.
- **Undo:** remove the `{'Name': 'yevgen-devbox'}` entry from the policy's TargetTags (same `update_lifecycle_policy` call).
- **Notes:** Instance has `StopWhenIdle=true`; DLM snapshots stopped instances fine (crash-consistent). Complements the git layer: `~/.claude` whitelist repo (init 2026-07-03) + `docs/` repo pushes.
