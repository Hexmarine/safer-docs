# Deployment Pipeline — sensor-alarm-backend (prod)

**Last updated:** 2026-04-12  
**Status:** Active; last successful deployment 2025-08-15

---

## Overview

The production API is deployed via a fully managed AWS CI/CD pipeline:

```
GitHub (SensorGlobal/sensor-alarm-backend, master branch)
  └─► CodePipeline (Smoke-API)
        ├─ Source  → CodeStar Connection pulls ZIP on master push
        ├─ Build   → CodeBuild (Smoke-API project) compiles TypeScript → JS
        └─ Deploy  → CodeDeploy (Blue/Green) → Auto Scaling Group (4–5 × t3.medium)
```

---

## Stage-by-stage breakdown

### 1. Source
- **Provider:** CodeStar Connections (`aeb983d6-b6f5-411d-b6b8-578756a127fc`)
- **Repo:** `SensorGlobal/sensor-alarm-backend`
- **Trigger branch:** `master`
- A push to `master` automatically starts the pipeline. No manual trigger needed.

### 2. Build — CodeBuild project `Smoke-API`
- **Image:** `aws/codebuild/standard:7.0` (Node.js 20)
- **Compute:** `BUILD_GENERAL1_SMALL`
- **Buildspec** (inline, not in repo — lives in CodeBuild console):
  1. Pulls `.env` from **S3: `s3://smokeapienv/env`** → renames to `.env`
  2. Installs TypeScript 4.9.5 globally
  3. `npm install --legacy-peer-deps` (removes `.husky` and `package-lock.json` first)
  4. `tsc --skipLibCheck` — compiles `src/` → `build/`
  5. Artifacts: all files (`**/*`)
- **Output artifact:** uploaded to `s3://codepipeline-ap-southeast-2-85960498242/Smoke-API/BuildArtif/`

### 3. Deploy — CodeDeploy `Smoke-API`
- **Deployment type:** Blue/Green with traffic control (`WITH_TRAFFIC_CONTROL`)
- **Deployment config:** `CodeDeployDefault.OneAtATime`
- **Target:** Auto Scaling Group `CodeDeploy_Smoke-API_d-NNRF41DGD`
- **ASG size:** min 4, max 5, desired 4 × t3.medium
- **AppSpec** (`appspec.yml` in repo):
  - Copies all files to `/home/ec2-user/smoke_api/` (overwrite)
  - `AfterInstall`: `scripts/install-dependencies.sh` (currently only runs puppeteer install; `npm install` is commented out)
  - `ApplicationStart`: `scripts/start-server.sh` — restarts PM2: `pm2 delete all` then `pm2 start build/index.js`

### 4. EC2 launch (Auto Scaling)
- **Launch template:** `prod-api-server-lt` (version 650, created 2023-07-01)
- **AMI:** `ami-06e5b001a75b3375f`
- **Instance type:** t3.medium
- **IAM profile:** `SSM_For_EC2` (enables SSM Session Manager access, no SSH keys needed)
- When ASG launches a replacement instance, CodeDeploy automatically re-deploys the last successful artifact to it (the Nov 14 2025 deployment was triggered this way — it was an autoscaling launch event, not a human push).

---

## Current state (as of 2026-04-12)

| Item | Value |
|---|---|
| Last pipeline run | **2025-08-15** (commit `dbe81521`, SENS-6487) |
| Code version on prod | Jan 2024 (`qa` branch HEAD, despite master being the trigger) |
| Build artifact on S3 | `Smoke-API/BuildArtif/br86wvJ` (built 2025-08-15) |
| Last CodeDeploy to humans | **2025-11-14** (triggered by autoscaling launch) |
| Servers running | 4 × prod-api-server (t3.medium) |

> **Note:** The servers show `git log` HEAD at `qa` branch / Jan 2024 even though the pipeline triggers on `master`. This likely means the build artifact is compiled from master (Aug 2025 code is in `build/`) but the git working tree on the server reflects the source that was copied verbatim — the `qa` HEAD is the branch the repo was last at locally. The `build/index.js` timestamp (2025-08-15) matches the last pipeline run.

---

## March 26, 2026 — server reboots (NOT a deployment)

The March 26 event that coincides with the SMS/alarm drop was **not a code deployment**:

| Finding | Detail |
|---|---|
| Last pipeline run | 2025-08-15 — 7 months before the incident |
| Server boot times | `i-09acc276e638ba099` rebooted 2026-03-26 23:49 UTC; `i-0b53bd5e7a9b9dec0` at 23:15 UTC; `i-0bcc2dcab7c6dfedf` at 2026-03-27 00:18 UTC |
| Trigger | Unknown — likely OS-level patch/reboot (kernel version shows `6.1.106-116.188`) |
| Code change | None — same build artifact re-deployed by CodeDeploy autoscaling hook on each restart |

The servers rebooted on March 26, and CodeDeploy's autoscaling hook re-applied the same Nov 2025 artifact. The SMS drop therefore coincides with reboot timing but was likely caused by the 4,808-row `tbl_alarms` bulk update on March 25–26 (separate DB investigation — see `2026-04-12-mysql-sms-root-cause.md`).

---

## env management

- Production `.env` lives at **S3: `s3://smokeapienv/env`** (pulled fresh on every build)
- CodeBuild `cat .env` in the buildspec means the full env is printed to CodeBuild logs — treat these logs as sensitive
- The `.env` on the EC2 servers (`/home/ec2-user/smoke_api/.env`) is a stub (`NODE_ENV=production\nSECRET_NAME=sensor-prod`) — the running process uses the secret loaded at startup from Secrets Manager secret `sensor-prod`
- `SYNCDB=no` in the secret — Sequelize `sync` is disabled; schema changes must be applied manually

---

## Other pipelines (not prod backend)

| Pipeline | Branch | Target |
|---|---|---|
| `Smoke-development-API` | development | dev EC2 |
| `Smoke-qa-API` | qa | QA EC2 (2 × t2.micro) |
| `Smoke-Sandbox-API` | — | sandbox |
| `uat-api` | — | UAT |
| `sso-api` | — | prod SSO (separate `prod-sso-api` server) |
| `Smoke-production-admin` | — | Angular admin frontend (S3/CloudFront) |
| `Smoke-API` | **master** | **prod backend (this doc)** |

---

## Key risks / migration notes

1. **`.env` in S3** — `s3://smokeapienv/env` must be recreated in the new account with updated values. The bucket is not public; ensure the new CodeBuild role has `s3:GetObject` on it.
2. **Buildspec inline** — The buildspec lives in CodeBuild console, not the repo. Must be manually copied when recreating the pipeline in a new account.
3. **`npm install` commented out** — `install-dependencies.sh` does NOT re-run `npm install` on deploy; it only runs `node install.mjs` for puppeteer. `node_modules` comes from the build artifact ZIP. This means the node_modules on production were last regenerated in August 2025.
4. **`SYNCDB=no`** — No automatic migrations on deploy. DB schema changes are manual.
5. **No rollback mechanism** — Blue/green CodeDeploy could roll back, but the `scripts/start-server.sh` currently doesn't handle startup failure gracefully (PM2 deletes all then starts, no health check before traffic shift).
6. **`start-server.sh` noise** — Contains `Restart logs rotate` and `pm2 LOGS ROTATE` as bare text (not comments or commands), which are ignored by bash but indicate the script is ad-hoc and unmaintained.
