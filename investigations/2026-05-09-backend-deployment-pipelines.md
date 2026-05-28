# Backend Deployment Pipeline Inventory

**Date:** 2026-05-09  
**AWS account:** `747293622182`  
**Region:** `ap-southeast-2`  
**Scope:** `sensor-alarm-backend`, related SSO pipelines, and deployment targets  
**Method:** Read-only AWS CLI plus repo inspection

---

## Summary

The production backend is deployed through the legacy AWS managed pipeline:

```text
GitHub SensorGlobal/sensor-alarm-backend master
  -> CodePipeline Smoke-API
  -> CodeBuild project Smoke-API
  -> CodeDeploy application/group Smoke-API / Smoke-API
  -> CodeDeploy-managed Auto Scaling Group CodeDeploy_Smoke-API_d-NNRF41DGD
  -> EC2/PM2 process behind target group SmokeAPI
```

The pipeline still exists and its GitHub CodeStar connection is `AVAILABLE`, but the latest production pipeline execution is old:

```text
2025-08-15 11:54-12:22 AEST
commit dbe81521260db18fc17d72a65a33dcf5287de99a
status Succeeded
```

Current live CodeDeploy-managed running app-tier ASGs are:

| ASG | Min | Max | Desired | Instances | Launch template |
|---|---:|---:|---:|---:|---|
| `CodeDeploy_Smoke-API_d-NNRF41DGD` | 2 | 5 | 2 | 2 | `prod-api-server-lt` |
| `CodeDeploy_sso-api-dg_d-QKLSZP50D` | 1 | 2 | 1 | 1 | `prod-sso-api-lt` |

---

## Production Backend Pipeline

### CodePipeline `Smoke-API`

Stages:

| Stage | Provider | Key config |
|---|---|---|
| Source | CodeStarSourceConnection | `SensorGlobal/sensor-alarm-backend`, branch `master`, output `CODE_ZIP` |
| Build | CodeBuild | project `Smoke-API` |
| Deploy | CodeDeploy | application `Smoke-API`, deployment group `Smoke-API` |

Artifact bucket:

```text
codepipeline-ap-southeast-2-85960498242
```

Source connection:

```text
arn:aws:codestar-connections:ap-southeast-2:747293622182:connection/aeb983d6-b6f5-411d-b6b8-578756a127fc
```

The same connection is `AVAILABLE`. Two older `smokealarm` GitHub connections also exist in `PENDING` state.

### CodeBuild `Smoke-API`

The project source is `NO_SOURCE` because CodePipeline passes the source artifact. The buildspec is inline in CodeBuild, not in the repo.

Important buildspec behavior:

- Uses image `aws/codebuild/standard:7.0` with Node.js 20.
- Copies `s3://smokeapienv/env` into `.env`.
- Runs `cat .env`, which prints environment contents into CodeBuild logs.
- Installs `typescript@4.9.5` globally.
- Runs `npm install --legacy-peer-deps --verbose`.
- Runs `tsc --skipLibCheck`.
- Publishes all files as the build artifact.

This means the buildspec and S3 env object are operational source-of-truth items that are not fully represented in the application repo.

### CodeDeploy `Smoke-API`

Deployment group details:

| Field | Value |
|---|---|
| Deployment type | `BLUE_GREEN` |
| Traffic control | `WITH_TRAFFIC_CONTROL` |
| Deployment config | `CodeDeployDefault.OneAtATime` |
| Green fleet | `COPY_AUTO_SCALING_GROUP` |
| Target group | `SmokeAPI` |
| Current ASG | `CodeDeploy_Smoke-API_d-NNRF41DGD` |
| Auto rollback | enabled for `DEPLOYMENT_FAILURE` |
| Current target revision | `s3://codepipeline-ap-southeast-2-85960498242/Smoke-API/BuildArtif/br86wvJ` |

Latest pipeline-created deployment:

```text
d-NNRF41DGD
created 2025-08-15 12:01 AEST
completed 2025-08-15 12:22 AEST
status Succeeded
```

Later CodeDeploy deployments on 2025-11-14 were created by Auto Scaling, not by new pipeline source revisions. They redeployed the same artifact `Smoke-API/BuildArtif/br86wvJ`.

---

## AppSpec And PM2 Startup

Repo file:

```text
code/sensor-alarm-backend/appspec.yml
```

CodeDeploy copies the artifact to:

```text
/home/ec2-user/smoke_api
```

Hooks:

| Hook | Script | Behavior |
|---|---|---|
| `AfterInstall` | `scripts/install-dependencies.sh` | runs Puppeteer `node install.mjs`; npm install is commented out |
| `ApplicationStart` | `scripts/start-server.sh` | deletes all PM2 processes, starts `build/index.js`, restarts awslogsd, installs PM2 logrotate |

Risk notes:

- `scripts/start-server.sh` contains `NODE_ENV=dev pm2 start build/index.js ...`, even though production runtime also uses `.env`/Secrets Manager loading. Treat this as fragile and verify runtime env directly before assuming branch/topic/environment behavior.
- The script includes bare text lines such as `Restart logs rotate` and `pm2 LOGS ROTATE`; they are not comments.
- There is no explicit health gate in the script before CodeDeploy traffic shift.

---

## Available Pipelines

### Backend API pipelines

| Pipeline | Repo | Branch | Build project | Deploy target | Latest execution |
|---|---|---|---|---|---|
| `Smoke-API` | `SensorGlobal/sensor-alarm-backend` | `master` | `Smoke-API` | `Smoke-API` / `Smoke-API` | 2025-08-15 succeeded |
| `Smoke-development-API` | `SensorGlobal/sensor-alarm-backend` | `development` | `Smoke-development-API` | `Smoke-development-API` / `Smokealarmdevelopmentapi` | 2025-07-30 succeeded |
| `Smoke-qa-API` | `SensorGlobal/sensor-alarm-backend` | `qa` | `Smoke-qa-API` | `Smoke-qa-API` / `Smoke-qa-API` | 2025-08-14 succeeded |
| `Smoke-Sandbox-API` | `SensorGlobal/sensor-alarm-backend` | `master` | `Smoke-Sandbox-API` | `Smoke-Sandbox-API` / `Smoke-Sandbox-Api-Dg` | 2025-08-15 succeeded |
| `uat-api` | `SensorGlobal/sensor-alarm-backend` | `uat` | `sensor-uat-api-codebuild` | `sensor-uat-api-codedeploy` / `uat-api-deployment-group` | no execution summary returned |

### SSO pipelines

| Pipeline | Repo | Branch | Build project | Deploy target | Latest execution |
|---|---|---|---|---|---|
| `sso-api` | `SensorGlobal/sso-provider` | `main` | `sso-api` | `sso-api` / `sso-api-dg` | 2025-07-21 succeeded |
| `dev-sso-api` | `SensorGlobal/sso-provider` | `development` | `dev-sso-api` | `dev-sso-api-dg` / `dev-sso-api-dg` | 2025-07-22 succeeded |
| `qa-sso-api` | `SensorGlobal/sso-provider` | `qa` | `qa-sso-api` | `qa-sso-api` / `qa-sso-api-dg` | 2025-07-22 succeeded |
| `sandbox-sso-api` | `SensorGlobal/sso-provider` | `main` | `sandbox-sso-api` | `sandbox-sso-api` / `sandbox-sso-api-dg` | 2025-07-21 succeeded |
| `uat-sso` | `SensorGlobal/sso-provider` | `uat` | `uat-sso-codebuild` | `uat-sso-codedeploy` / `uat-sso-dg` | no execution summary returned |

Other CodePipeline entries also exist for admin frontends and Odoo addons:

- `Smoke-production-admin`
- `Smoke-development-Admin`
- `Smoke-qa-Admin`
- `smoke-sandbox-admin`
- `smoke-uat-admin`
- `Odoo-Addons`

---

## Key Operational Risks

1. The production backend pipeline is real but stale; latest source-triggered prod backend execution was 2025-08-15.
2. CodeBuild prints `.env` to logs, so build logs should be treated as sensitive.
3. Buildspec is inline in CodeBuild and must be exported/recreated explicitly during migration.
4. The deployment depends on `s3://smokeapienv/env`, CodeStar GitHub connection state, CodeDeploy hooks, and a CodeDeploy-managed ASG.
5. `install-dependencies.sh` does not run `npm install` on the EC2 host; runtime `node_modules` come from the build artifact.
6. Current active prod API capacity is two EC2 instances, not the older four-instance baseline.
7. Non-prod pipelines still exist even where active non-prod app ASGs may have been stopped or decommissioned.

---

## Recommended Follow-Up

1. Export and preserve the inline CodeBuild buildspecs for prod API and SSO.
2. Export the S3 env object metadata and contents through an approved secret-handling path.
3. Confirm whether GitHub pushes to `master` still trigger `Smoke-API` automatically by checking CodeStar webhook/connection behavior before relying on it.
4. Verify runtime environment variables on a current prod API instance without printing secrets, especially `NODE_ENV`, `SECRET_NAME`, and PM2 process metadata.
5. Decide whether future backend changes should continue through this legacy CodePipeline path or move to the newer EKS/container path.
