# Repo & Operations Reference

Operational reference for working in this repository: repo layout, per-project
build commands, platform architecture, AWS access mechanics, and investigation
conventions.

> Guardrails, approval policy, and the AWS read-only posture are defined in
> `AGENTS.md` — the authoritative ruleset. This document is reference material
> only; it does not define rules. Where the two could appear to differ,
> `AGENTS.md` wins.

## Repo shape

- There is **no root build/test/lint workflow**. Run commands inside the
  relevant project under `code/`.
- The repo mixes investigation docs with snapshots of several application repos
  and utilities.
- Main runtime components:
  - `code/sensor-alarm-backend` — main Node/Express backend API
  - `code/sso-provider` — separate OIDC / SSO service
  - `code/sensor-angular` — Angular web portals
  - `code/mqtt-prototype` — .NET hub/controller simulator for MQTT validation
  - `code/odoo-*` — Odoo customizations and supporting material

## Platform architecture

- The production platform is an **IoT operations system**, not just a CRUD web
  app:
  1. field alarm/sensor event
  2. local Sensor Hub over RF
  3. hub backhaul over LTE Cat-M / KORE SIM
  4. MQTT broker
  5. Node/Express backend on EC2
  6. MySQL + MongoDB + Kafka + Redis supporting state and logs
  7. Angular portals, mobile apps, Odoo, and external integrations
- The backend subscribes to shared MQTT response topics and publishes commands
  back to per-hub command topics. See
  `docs/investigations/2026-04-09-platform-architecture-and-data-flow.md` and
  `code/sensor-alarm-backend/src/services/mqtt/subscriber.ts`.
- Public web surfaces are split across Angular portals, API/SSO ALBs, and
  CloudFront/S3-hosted frontends; keep that separation in mind when tracing a
  flow.

## Project commands

### `code/sensor-alarm-backend`

- Install: `npm install`
- Build: `npm run build`
- Lint: `npm run lint`
- Lint fix: `npm run lint:fix`
- Test: `npm test`
- Single test file:
  ```bash
  npm test -- test/path/to/file.test.ts
  ```
- Dev server: `npm run dev`
- CI-style local run: `npm run ci`

### `code/sensor-angular`

- Install: `npm install`
- Dev server: `npm start`
- Build: `npm run build`
- Lint: `npm run lint`
- Test: `npm test`
- Single spec:
  ```bash
  npm test -- --include src/app/path/to/file.spec.ts
  ```
- Environment switching scripts exist and are important for
  investigations/build prep:
  - `npm run set-dev-env`
  - `npm run set-qa-env`
  - `npm run set-sandbox-env`
  - `npm run set-prod-env`

### `code/sso-provider`

- Install: `npm install`
- Build: `npm run build`
- Start: `npm start`
- `npm test` is a placeholder that exits with an error; do not treat it as a
  real test suite.

### `code/mqtt-prototype`

- Requires .NET 7
- Run:
  ```bash
  dotnet run
  ```
- Use this simulator when validating broker behavior or hub command/response
  flows during migration work.

## AWS access and remote investigation

- Prefer **AWS Systems Manager Session Manager** over OpenVPN, SSH keys, or
  PEM-based copy flows when accessing instances.
- If AWS CLI auth is expired, refresh it from a local terminal with:
  ```bash
  bash scripts/aws-mfa-login.sh default sensorsyn-mfa
  ```
- Preferred session pattern:
  ```bash
  AWS_PROFILE=sensorsyn-mfa aws ssm start-session --region ap-southeast-2 --target <instance-id>
  ```
- When giving commands to the user, make them **copy-pasteable** and label the
  execution context explicitly:
  - **local terminal**
  - **inside SSM session**
  - **inside the repo / specific package directory**
- Do not assume remote instances have extra tools installed (`aws`, `jq`, local
  SSH keys, etc.). Check or suggest fallback commands that use standard shell
  tooling.

## Terraform workflow conventions

> Whether `terraform apply` (or any infrastructure mutation) may be run at all
> is governed by `AGENTS.md`. The conventions below apply to Terraform work
> performed within that policy.

- **Import safety — check for duplicate ownership first**: Before importing any
  resource into an env module, run `terraform state list` across all sibling
  modules to confirm the resource is not already tracked elsewhere. Resources of
  type `aws_acm_certificate`, `aws_route53_zone`, `aws_route53_record`,
  `aws_iam_role`, `aws_iam_instance_profile`, and
  `aws_iam_role_policy_attachment` belong exclusively in the `account/` module —
  never import them into `prod/`, `dev/`, `qa/`, or `sandbox/`.
- **Destroy guard**: After every `terraform plan`, check the output for any
  `destroy` operations. If any resource is marked for destruction, stop
  immediately. Diagnose the root cause (typically: a `.tf` resource block exists
  without a matching state entry, or vice versa) and resolve it before
  proceeding. This environment contains live production resources; a mistaken
  destroy requires manual AWS recovery.
- **ALB cert rotation — clear expired SNI entries**: When replacing a TLS
  certificate on an ALB listener, after attaching the new cert always check the
  listener's SNI certificate list for expired certs and remove them. ALB
  evaluates SNI matches against all listed certs in order; an expired cert that
  still matches the SNI hostname will be served instead of the new cert until it
  is explicitly removed.

## Investigation conventions

- For EMQX investigations, prefer **live node inspection** (`emqx_ctl`,
  `emqx eval`) before unpacking exports. EMQX export artifacts contain **binary
  Mnesia data**, not human-readable JSON.
- For manual investigations on remote hosts, prefer the smallest command that
  proves the next fact, then refine. Avoid long speculative command chains when
  you have not yet confirmed the file format or tool availability.
- When documenting findings from AWS or SSM work, capture the exact resource
  names, ports, listener types, and account/region context in the investigation
  note instead of leaving them implicit.
