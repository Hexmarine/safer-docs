# Backend Pipeline Validation And Versioning

- Date: 2026-05-09
- Scope: `code/sensor-alarm-backend`, production `Smoke-API` deployment path
- Related systems: GitHub `SensorGlobal/sensor-alarm-backend`, CodePipeline `Smoke-API`, CodeBuild `Smoke-API`, CodeDeploy `Smoke-API/Smoke-API`, target group `SmokeAPI`

## Objective

Identify the lowest-risk production pipeline validation change, summarize historical backend development patterns, and record the current versioning/release-identification state.

## Sources Used

- Repository files:
  - `code/sensor-alarm-backend/README.MD`
  - `code/sensor-alarm-backend/package.json`
  - `code/sensor-alarm-backend/appspec.yml`
  - `code/sensor-alarm-backend/scripts/start-server.sh`
  - `code/sensor-alarm-backend/.github/workflows/*`
- Local git history:
  - `git log --oneline --decorate`
  - `git log --merges`
  - `git log --graph --decorate --all`
  - `git branch -r`
  - `git tag`
- AWS read-only checks:
  - `aws deploy get-deployment-group --application-name Smoke-API --deployment-group-name Smoke-API`
  - `aws elbv2 describe-target-groups --names SmokeAPI`
  - `aws elbv2 describe-target-health --target-group-arn ...`

## Findings

The recommended first production pipeline validation is a documentation-only commit to the backend repository `master` branch. This should exercise the full live deployment path without changing runtime behavior:

`GitHub master -> CodePipeline Smoke-API -> CodeBuild Smoke-API -> CodeDeploy Smoke-API -> blue/green EC2 deployment behind target group SmokeAPI`

The production deployment group currently uses CodeDeploy blue/green deployment with traffic control and `CodeDeployDefault.OneAtATime`. Auto rollback is enabled for deployment failure events, but CloudWatch alarm rollback is not configured. The target group `SmokeAPI` currently has two healthy instance targets.

A docs-only commit is still a real production deployment. CodeDeploy copies the whole artifact to `/home/ec2-user/smoke_api` and restarts the PM2 process through `scripts/start-server.sh`.

Historical development was PR-heavy and Jira-ticket-driven. Recent commits and branch names commonly use:

- `feature/SENS-...`
- `bug/SENS-...`
- `hotfix-...`
- `sprint-...`
- environment branches such as `development`, `qa`, and `master`

Recent merge history shows individual feature or bug branches being merged through lower environments and then into `master`. For example, `feature/SENS-6487-remove-default-job-notes` was merged to `qa` and then to `master`. Sprint and hotfix branches were also used for production promotion.

The local remote-branch snapshot contains many long-lived historical branches:

- feature branches: 289
- bug branches: 106
- hotfix branches: 32
- development-style branches: 13
- qa-style branches: 24
- sprint branches: 35
- other branches: 30

Recent commit messages often include ticket IDs and time tracking text, for example `SENS-6487 #time 15m ...`. In the latest 300 commit subjects checked, 98 were PR merge commits, 180 started with `SENS-`, and 118 mentioned `#time`.

## Versioning And Release Identification

There is no evidence of formal release tags in the local backend repo; `git tag` returned no tags.

`package.json` contains `"version": "1.0.0"`, but this does not appear to be used as a deployment release version. `appspec.yml` uses CodeDeploy schema `version: 0.0`, not an application version.

The deployed production identity is currently best represented by:

- Git commit SHA from the `master` source action
- CodePipeline execution
- CodeBuild build execution
- CodeDeploy deployment ID
- S3 artifact key in the pipeline artifact bucket

The app initializes Sentry with `SENTRY_ENV`, but no Sentry `release` value or Git SHA is configured in code. There is also no obvious runtime endpoint that returns the deployed commit/build.

The repository has GitHub Actions for GitGuardian scanning and a custom merge-request check. The actual production build/deploy is not driven by GitHub Actions; it is driven by AWS CodePipeline/CodeBuild/CodeDeploy. The CodeBuild project installs dependencies with `npm install --legacy-peer-deps` from the pipeline artifact. No committed `package-lock.json`, `yarn.lock`, or `pnpm-lock.yaml` is present in the deployed commit.

## Risks Or Constraints

- A docs-only commit validates deployment mechanics but does not prove a new runtime code path is being served.
- The PM2 start hook restarts the application and log agent during deployment, so even a no-op deploy is production-impacting.
- The target group health check accepts HTTP `200-499`, so it is useful for instance reachability but weak as an application correctness gate.
- Lack of immutable dependency lockfile means rebuilds may not be byte-for-byte reproducible.
- Lack of release metadata in Sentry/runtime responses makes post-deploy verification depend on AWS deployment metadata rather than the service itself.

## Recommended Next Steps

1. Open a docs-only PR against `master` in `SensorGlobal/sensor-alarm-backend`.
2. Merge during a quiet window and watch `Smoke-API` Source, Build, and Deploy stages.
3. Verify CodeDeploy success and `SmokeAPI` target health after deployment.
4. After the no-op deploy succeeds, add a small runtime build/version signal in a separate PR.
5. Consider adding release metadata to Sentry and a dependency lockfile once ownership norms are settled.

## Open Questions

- Should production promotion continue to be direct merges to `master`, or should a documented `development -> qa -> master` promotion rule be reinstated?
- Should runtime version identity be exposed through the existing IP-protected monitoring route or through a separate lightweight endpoint?
- Should CodeBuild keep using floating dependency resolution, or should the backend adopt a committed npm lockfile?
