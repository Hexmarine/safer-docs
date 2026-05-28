# Local K8s Validation Backlog

- Created: 2026-04-29
- Purpose: track the local Kubernetes and MQTT simulator validation work for the EKS/EMQX migration stream.
- Implementation repo: `code/infra/k8s` (local harness scaffolded; no remote deployment).
- Source docs:
  - `docs/infra/kubernetes-migration.md`
  - `docs/investigations/2026-04-29-eks-platform-migration-plan.md`
  - `docs/investigations/2026-04-09-mqtt-device-communications.md`
  - `code/mqtt-prototype/README.md`
- Change log rule: this backlog is for local tooling, harnesses, and tests only. Do not record local-only work in `docs/applied-changes.md`.
- Infra editing gate: writing manifests, scripts, or CI files under `code/infra/k8s` needs explicit approval because it is an infra repository. Documentation updates here are safe by default.

## Goal

Build a repeatable local validation loop that can run most of the SensorSyn product shape in `kind` before production EKS work begins.

The first useful loop is:

1. Run a local Kubernetes cluster with `kind`.
2. Run EMQX inside that cluster.
3. Expose EMQX to the host through `kubectl port-forward`.
4. Run the controller/hub simulator from `code/mqtt-prototype` outside the cluster.
5. Run automated scenarios that publish server-side MQTT commands and assert simulator responses.

This deliberately kept `sensor-alarm-backend` out of the first slice. That milestone validated broker deployment, topic behavior, simulator automation, and repeatable test execution without MySQL, MongoDB, Redis, Kafka-compatible messaging, API secrets, or backend containerization.

The next loop should move toward a locally validatable product slice:

1. Run EMQX, API, SSO, workers/jobs, and local test dependencies inside `kind`.
2. Use Kubernetes namespaces to keep each component boundary visible.
3. Use Redpanda as the local Kafka-compatible dependency when async/audit messaging needs validation.
4. Keep production databases and SaaS services represented by local fixtures, fakes, or explicit bypasses.
5. Keep every validation path one-command, deterministic, and free of AWS calls or production secrets.

## Design Defaults

| Area | Default |
|---|---|
| Implementation repo | `code/infra/k8s` |
| Local cluster | `kind` |
| Broker | EMQX in Kubernetes |
| Backend API | `sensor-alarm-backend` in Kubernetes after host-run stability |
| Auth | `sso-provider` in Kubernetes after backend app deployment |
| Background jobs | Kubernetes `CronJob`s or worker Deployments |
| Kafka-compatible local dependency | Redpanda in Kubernetes, local-only |
| Local data dependencies | MySQL, MongoDB, Redis, and Redpanda fixtures in Kubernetes |
| Simulator | `code/mqtt-prototype`, running outside Kubernetes |
| Simulator runtime | host `dotnet` first; Kubernetes Job later if useful |
| Broker exposure | `kubectl port-forward` to localhost for v1 |
| MQTT ports | `1883` and production-like `18756` |
| Dashboard port | `18083` |
| Secrets | local test credentials only; no production secrets |
| First assertion layer | MQTT test driver, then backend-visible DB/API assertions |

## Status Legend

| Status | Meaning |
|---|---|
| `Proposed` | Item is scoped but not started. |
| `Ready` | Enough design detail exists to implement. |
| `In progress` | Work has started locally. |
| `Blocked` | Waiting on a tool, dependency, or decision. |
| `Done` | Implemented and verified locally. |
| `Deferred` | Parked for a later phase. |

## Current Focus

`LK8S-024 Local validation documentation refresh`

## Backlog

### LK8S-001 - Toolchain bootstrap

- **Status:** `Done`
- **Priority:** P1
- **Purpose:** make the local machine capable of running the MQTT validation loop.
- **Current evidence:** Docker, `kubectl`, `kind`, and .NET SDK `8.0.126` are installed. Docker daemon and `kubectl` client checks passed when run outside the sandboxed shell. `helm`, `mosquitto_pub`, and `mosquitto_sub` are missing but remain optional for v1.
- **Recommended action:** document and/or install the minimum tools for v1:
  - `kind`
  - .NET SDK compatible with `code/mqtt-prototype` target `net7.0`
  - optional MQTT CLI client for manual inspection, such as `mosquitto_pub` / `mosquitto_sub` or MQTTX
- **Acceptance criteria:**
  - `docker` works.
  - `kubectl version --client` works.
  - `kind version` works.
  - `dotnet --version` works and can build `code/mqtt-prototype`.
- **Verification command candidates:**
  - `docker ps`
  - `kubectl version --client`
  - `kind version`
  - `dotnet build code/mqtt-prototype/controller-prototype.csproj`
- **Notes:** `helm` is deferred unless the first EMQX deployment uses the official chart. A raw manifest is enough for the first local broker proof.
- **Verification 2026-04-29:** `./scripts/check-tools.sh` passed for required tools. `code/mqtt-prototype` was retargeted to `net8.0`; `dotnet build code/mqtt-prototype/controller-prototype.csproj` succeeded with remaining warnings for known `SixLabors.ImageSharp` `3.1.3` vulnerabilities.

### LK8S-002 - Local kind cluster

- **Status:** `Done`
- **Priority:** P1
- **Purpose:** create an isolated local Kubernetes cluster for MQTT validation.
- **Recommended action:** keep using the scaffolded cluster path in `code/infra/k8s`.
- **Implementation target:** local automation now provides:
  - `up`: create cluster if missing
  - `status`: show nodes, namespaces, EMQX pod/service status
  - `down`: delete cluster
- **Acceptance criteria:**
  - `kind create cluster --name sensorsyn-mqtt` succeeds.
  - `kubectl config current-context` points at `kind-sensorsyn-mqtt`.
  - `kubectl get nodes` shows a ready node.
  - deleting and recreating the cluster is repeatable.
- **Notes:** multi-node kind is not required for the first loop. Add multi-node only when testing EMQX clustering or disruption behavior.
- **Verification 2026-04-29:** `./scripts/up.sh` created local kind cluster `sensorsyn-mqtt`; `./scripts/status.sh` confirmed node `sensorsyn-mqtt-control-plane` is `Ready`.

### LK8S-003 - Local EMQX deployment

- **Status:** `Done`
- **Priority:** P1
- **Purpose:** run an EMQX broker inside the local cluster that behaves enough like production MQTT for protocol tests.
- **Recommended action:** use the scaffolded single-node EMQX manifest from `code/infra/k8s`; add chart/operator variants later.
- **Required local listener behavior:**
  - MQTT TCP on `1883`
  - production-like MQTT TCP on `18756`
  - dashboard/API on `18083`
- **Default exposure:** use `kubectl port-forward` from service or pod to host:
  - `localhost:1883`
  - `localhost:18756`
  - `localhost:18083`
- **Acceptance criteria:**
  - EMQX pod reaches ready state.
  - Host can connect to `localhost:18756`.
  - Dashboard is reachable at `http://localhost:18083` for local credentials.
  - A manual publish/subscribe smoke test succeeds through the forwarded port.
- **Notes:** production TLS, CIDR restrictions, NLB behavior, and operator lifecycle are out of v1. They belong in later local or cloud-like validation.
- **Verification 2026-04-29:** `./scripts/up.sh` deployed EMQX `5.8.7`; pod `emqx-5484f8f46d-scwh6` reached `1/1 Running`. `./scripts/port-forward.sh` exposed `1883`, `18756`, and `18083`; dashboard returned HTTP `200`; raw MQTT CONNECT smoke checks returned CONNACK accepted on both `1883` and `18756`.

### LK8S-004 - Simulator non-interactive mode

- **Status:** `Done`
- **Priority:** P1
- **Purpose:** make `code/mqtt-prototype` usable in automated integration tests.
- **Current evidence:** the prototype already connects as an MQTT 3.1.1 client, subscribes to `sg/sas/cmd/{controllerSerial}`, publishes to `sg/sas/resp/{controllerSerial}`, and has methods for key command responses.
- **Recommended action:** add non-interactive modes while preserving the existing menu mode:
  - `simulator`: start controller(s), subscribe to command topics, and keep running until interrupted.
  - `scenario`: run a scenario file, publish/receive messages, assert expectations, and exit.
- **Proposed command shapes:**
  - `dotnet run -- --mode simulator --config config.local.json`
  - `dotnet run -- --mode scenario --scenario scenarios/verify.json --config config.local.json`
- **Acceptance criteria:**
  - Existing `dotnet run` menu behavior still works.
  - `--mode simulator` starts without interactive prompts.
  - Simulator exits cleanly on `SIGINT` / Ctrl+C.
  - Invalid config exits non-zero with a clear error.
- **Notes:** the simulator should continue to live outside Kubernetes for v1. This mirrors a real hub/device connecting into the broker from outside the cluster.
- **Verification 2026-04-29:** `code/mqtt-prototype` now supports `--mode simulator --config ...`, keeps the original menu behavior as the default, retargets to `net8.0`, and was smoke-tested against local EMQX. The simulator received `VERIFY` on `sg/sas/cmd/LOCALC001000000001` and published the expected response on `sg/sas/resp/LOCALC001000000001`.

### LK8S-005 - Scenario file format

- **Status:** `Done`
- **Priority:** P1
- **Purpose:** define the contract for repeatable MQTT integration tests.
- **Recommended action:** add JSON scenario files that describe controller setup, command publishes, expected responses, and timeouts.
- **Minimum scenario fields:**
  - scenario name
  - controller serial
  - initial alarms
  - MQTT command topic
  - MQTT command payload
  - expected response topic
  - expected response payload fields
  - timeout in milliseconds
- **Example intent:**
  - publish `{"CMD":"VERIFY"}` to `sg/sas/cmd/{controllerSerial}`
  - assert a response on `sg/sas/resp/{controllerSerial}`
  - assert `CMD=VERIFY`, `STATUS=1`, `CONTROLLERSERIAL={controllerSerial}`
- **Acceptance criteria:**
  - Scenario files are deterministic.
  - Timeouts fail tests quickly.
  - Assertion failures show expected vs actual payload.
  - No scenario requires production credentials or AWS access.
- **Verification 2026-04-29:** added ordered scenario files under `code/infra/k8s/scenarios/mqtt/` with scenario name, controller serial, initial alarms placeholder, command topic/payload, expected response topic/fields, and timeout.

### LK8S-006 - MQTT test driver

- **Status:** `Done`
- **Priority:** P1
- **Purpose:** provide a small automated client that drives scenarios against EMQX and the simulator.
- **Recommended action:** implement the first driver either inside `mqtt-prototype` scenario mode or as a small separate local test runner. Prefer reusing the prototype code if it stays simple.
- **Acceptance criteria:**
  - Driver can connect to `localhost:18756`.
  - Driver can publish server-side commands to `sg/sas/cmd/{controllerSerial}`.
  - Driver can subscribe/assert responses from `sg/sas/resp/{controllerSerial}`.
  - Driver exits `0` on success and non-zero on timeout/assertion failure.
- **Notes:** this v1 driver replaces manual MQTTX/VSQTT checks for repeatability.
- **Verification 2026-04-29:** added `code/infra/k8s/scripts/run-mqtt-scenarios.mjs`, a dependency-free MQTT scenario driver using Node's TCP client. `make test` built the simulator, started local port-forward, started simulator mode, ran `VERIFY`, `ALARMS`, `ADD`, `BATTERY`, `TEST`, `REMOVE`, and post-remove `ALARMS` scenarios, asserted response fields, and exited `0`.

### LK8S-007 - Automation wrapper

- **Status:** `Done`
- **Priority:** P2
- **Purpose:** provide a one-command local loop for developers and future CI.
- **Recommended action:** keep using the wrapper commands as the local development loop and CI baseline.
- **Target commands:**
  - `up`: create kind cluster and deploy EMQX
  - `test`: run all MQTT scenarios
  - `logs`: collect EMQX and simulator/test logs
  - `down`: delete kind cluster
- **Acceptance criteria:**
  - A clean machine can run the documented sequence.
  - Re-running `up` is idempotent enough for local use.
  - `down` leaves no running local cluster.
  - Failure output points to the failing phase.
- **Verification 2026-04-29:** `check`, `validate`, `up`, `status`, `port-forward`, `test`, and `down` have been exercised successfully. `down` deleted local cluster `sensorsyn-mqtt`; `up` recreated it from zero and redeployed EMQX; `status` confirmed the recreated node and EMQX pod were ready; `make test` passed the full MQTT scenario suite against the fresh cluster. The local cluster remains available for follow-up testing after recreation.

### LK8S-008 - Initial MQTT scenarios

- **Status:** `Done`
- **Priority:** P1
- **Purpose:** cover the current hub/controller command surface before backend integration.
- **Initial scenarios:**
  - `VERIFY`
  - `ALARMS`
  - `ADD`
  - `REMOVE`
  - `BATTERY`
  - `TEST`
  - `ALERT` on/off
  - `TAMPERED`
  - `DISCONNECT`
  - `RECONNECT`
  - `LOW_BATTERY`
- **Acceptance criteria:**
  - Each scenario can run independently.
  - Each scenario records the topics and payloads exchanged.
  - Scenario coverage maps back to the topic model in `docs/investigations/2026-04-09-mqtt-device-communications.md`.
- **Notes:** backend behavior gaps, such as discarded `LOW_BATTERY`, should be tested in the later backend integration phase, not in the broker-only phase.
- **Verification 2026-04-29:** automated coverage now includes `VERIFY`, `ALARMS`, `ADD`, `REMOVE`, `BATTERY`, `TEST`, `ALERT` on/off, `TAMPERED`, `DISCONNECT`, `RECONNECT`, and `LOW_BATTERY`. Server-command scenarios publish to `sg/sas/cmd/{controllerSerial}` and assert simulator responses from `sg/sas/resp/{controllerSerial}`. Device-event scenarios publish directly to `sg/sas/resp/{controllerSerial}` and assert broker delivery of the expected event payloads. `make test` passed end-to-end against local kind EMQX.

### LK8S-009 - Local backend integration prep

- **Status:** `Done`
- **Priority:** P2
- **Purpose:** identify the exact runtime, configuration, and service dependencies needed to run `sensor-alarm-backend` locally against the local EMQX loop.
- **Recommended action:** inspect backend startup, MQTT subscriber config, environment variables, health endpoints, and data-store clients before adding containers or scripts.
- **Acceptance criteria:**
  - Backend entrypoint and local run command are known.
  - Required environment variables are listed.
  - MQTT connection settings needed for local EMQX are known.
  - Required MySQL, MongoDB, Redis, Kafka-compatible messaging, and secrets behavior is classified as required, optional, or bypassable for local tests.
  - First backend-visible assertion target is chosen.
- **Notes:** this is a local discovery/planning item only; do not add CI or AWS dependencies here.
- **Verification 2026-04-29:** documented findings in `docs/investigations/2026-04-29-local-backend-integration-prep.md`. Backend starts from root `index.ts`; `NODE_ENV=local` bypasses Secrets Manager; startup still requires MySQL, MongoDB main/archive, Redis, MQTT, and socket wiring. Kafka is bypassable with `DONT_USE_KAFKA=true` for the first local loop; the local-product backlog now uses Redpanda for Kafka-compatible validation once messaging assertions are needed. First recommended backend-visible assertion is a `DISCONNECT` event updating local MySQL `tbl_alarms.connectedStatus`.

### LK8S-010 - Backend local runtime

- **Status:** `Done`
- **Priority:** P2
- **Purpose:** run `sensor-alarm-backend` locally against the local EMQX broker with the smallest viable service set.
- **Recommended action:** start with backend outside Kubernetes, EMQX inside kind, and backing services inside kind namespace `backend-deps-local`.
- **Acceptance criteria:**
  - Backend process starts from a documented command.
  - Backend connects to local EMQX on `localhost:18756` or `localhost:1883`.
  - Backend reaches a usable health/readiness state.
  - Startup uses local-only env/config and no production secrets.
  - Required backing services are either running locally or deliberately stubbed/bypassed.
- **Notes:** keep backend outside Kubernetes until the minimum runtime is understood. Move it into kind only after local process execution is stable. A local-only env fixture now exists at `code/infra/k8s/env/sensor-alarm-backend.local.env.example`.
- **Verification 2026-04-29:** `sensor-alarm-backend` built with `pnpm run build` and started through the local env fixture against forwarded EMQX, MySQL, MongoDB, and Redis. Runtime connected successfully to Redis, MySQL, MongoDB main/archive, and MQTT. Local-only app fixes disabled Kafka cleanly through `DONT_USE_KAFKA` and skipped LaunchDarkly initialization in `NODE_ENV=local`. Added `scripts/backend-start.sh` for repeatable startup.

### LK8S-011 - Local backing services

- **Status:** `Done`
- **Priority:** P2
- **Purpose:** provide repeatable local dependencies for the backend integration phase.
- **Recommended action:** use the local Kubernetes dependency manifests under `code/infra/k8s/manifests/backend-deps`.
- **Candidate services:**
  - MySQL fixture in namespace `backend-deps-local`
  - MongoDB fixture in namespace `backend-deps-local`
  - Redis Stack fixture in namespace `backend-deps-local`
  - Redpanda fixture in namespace `backend-deps-local` when Kafka-compatible paths are validated
  - Kafka-compatible messaging bypassed with `DONT_USE_KAFKA=true` until Redpanda is wired into the local loop
  - local Secrets Manager replacement via `NODE_ENV=local` and env fixture
- **Acceptance criteria:**
  - Services start from one documented local command.
  - Test fixtures are deterministic.
  - No production data or credentials are used.
  - Teardown removes local-only state or clearly documents persisted volumes.
- **Notes:** do not add full Kafka/ZooKeeper for local validation. Use Redpanda as the lightweight Kafka-compatible fixture when backend startup, audit/event assertions, or worker tests require a broker. The dependency manifests use `emptyDir` storage so data is local and disposable with the pod/cluster.
- **Verification 2026-04-29:** added `backend-deps-local` namespace with MySQL `8.0`, MongoDB `7.0`, and Redis Stack `7.2.0-v16`; added `deps-up`, `deps-status`, `deps-port-forward`, and `deps-logs` scripts plus `make up-all`. `backend-local` is now reserved for the future backend app namespace. `./scripts/deps-up.sh` deployed all three dependency services successfully; pods reached `1/1 Running`; localhost forwards opened on MySQL `13306`, MongoDB `27018`, and Redis `16379`; in-cluster checks confirmed MySQL database `smoke_alarm_local`, MongoDB ping `1`, and Redis `PONG`.

### LK8S-012 - Backend MQTT integration tests

- **Status:** `Done`
- **Priority:** P2
- **Purpose:** extend the current MQTT scenario suite to assert backend-visible effects.
- **Recommended action:** use `code/infra/k8s/scripts/backend-smoke-test.sh` for the first backend-visible smoke loop.
- **Acceptance criteria:**
  - Backend subscribes to local EMQX shared response topic.
  - Simulator or scenario runner publishes controller events.
  - Tests assert backend-visible effects, not only MQTT broker delivery.
  - Failure output identifies whether the break is broker, simulator, backend, or data-store related.
- **Notes:** the first assertion intentionally stops at backend-visible DB state. Broader notification-path assertions need more fixture tables.
- **Verification 2026-04-29:** added `sql/backend-local-smoke.sql`, `scripts/backend-db-init.sh`, and `scripts/backend-smoke-test.sh`. `./scripts/backend-smoke-test.sh` now brings up local dependencies, seeds the DB, starts EMQX and dependency port-forwards, starts the host-run backend, waits for MQTT readiness, publishes `scenarios/mqtt/012-disconnect.json`, asserts local MySQL child alarm `LOCALA001000000001` changes from `connectedStatus=1` to `connectedStatus=0`, then cleans up background backend and port-forward processes.

### LK8S-014 - Backend smoke hardening

- **Status:** `Proposed`
- **Priority:** P2
- **Purpose:** reduce noise and broaden confidence after the first backend-visible smoke test.
- **Recommended action:** harden the local backend fixture and test script before adding CI.
- **Candidate improvements:**
  - Add backend log assertions for absence of unexpected local runtime errors.
  - Expand the smoke fixture only as needed for the next event assertions.
  - Add `RECONNECT` as the second DB-state assertion.
  - Consider a local backend test mode that disables notification side effects cleanly.
  - Decide whether the backend app should move into namespace `backend-local` after host-run stability.
- **Acceptance criteria:**
  - The smoke test remains one-command and deterministic.
  - Failures distinguish startup, MQTT, and DB assertion phases.
  - Additional fixture data is documented and local-only.

### LK8S-015 - Backend image and in-cluster deployment

- **Status:** `Done`
- **Priority:** P1
- **Purpose:** move `sensor-alarm-backend` from a host-run process into namespace `backend-local` so the local stack looks like the future EKS product shape.
- **Recommended action:** add a local-only backend image build path and Kubernetes Deployment using the existing env fixture and in-cluster service names.
- **Acceptance criteria:**
  - Backend image builds from the checked-out repo without production secrets.
  - Backend Deployment runs in namespace `backend-local`.
  - Backend connects to EMQX, MySQL, MongoDB, and Redis through Kubernetes service DNS.
  - The existing backend smoke test can target either host-run or in-cluster backend mode.
  - Logs are readable through `kubectl logs` and startup failures are obvious.
- **Notes:** keep the first in-cluster backend replica count at `1` until MQTT/Kafka-compatible consumer scaling behavior is understood.
- **Verification 2026-04-29:** added a local-only backend image path, Kubernetes ConfigMap/Deployment/Service in namespace `backend-local`, and `backend-image-build`, `backend-up`, and `backend-status` scripts. `./scripts/backend-up.sh` built the backend, loaded `sensorsyn/sensor-alarm-backend:local` into kind, deployed one backend replica, and verified the pod reached `1/1 Running`. The in-cluster backend connected to EMQX, MySQL, MongoDB main/archive, and Redis through Kubernetes service DNS.

### LK8S-016 - In-cluster backend smoke test

- **Status:** `Done`
- **Priority:** P1
- **Purpose:** validate the API/MQTT/DB path without depending on host port-forwards for backend runtime.
- **Recommended action:** add a smoke wrapper that deploys the backend app, applies local fixtures, publishes MQTT scenarios, and asserts backend-visible effects from inside or against the cluster.
- **Acceptance criteria:**
  - One command brings up EMQX, backend dependencies, backend app, and test fixtures.
  - `DISCONNECT` and `RECONNECT` assertions pass against local MySQL.
  - The command exits non-zero with clear phase labels on failure.
  - Cleanup leaves no stray local backend or port-forward processes.
- **Verification 2026-04-29:** extended `scripts/backend-smoke-test.sh` with `SENSORSYN_BACKEND_MODE=cluster` and `make backend-smoke-test-cluster`. The cluster-mode smoke test rebuilds/redeploys the backend, seeds the local MySQL fixture, publishes `DISCONNECT` and `RECONNECT` through local EMQX, and passed by asserting child alarm `LOCALA001000000001.connectedStatus` changed `1 -> 0 -> 1`.

### LK8S-017 - Redpanda local dependency

- **Status:** `Done`
- **Priority:** P1
- **Purpose:** provide Kafka-compatible local validation without running full Kafka/ZooKeeper in `kind`.
- **Recommended action:** add Redpanda to `backend-deps-local` with local-only storage, service DNS, and optional host port-forwarding.
- **Acceptance criteria:**
  - Redpanda pod starts in namespace `backend-deps-local`.
  - Backend can run with Kafka-compatible settings pointed at Redpanda instead of `DONT_USE_KAFKA=true`.
  - A local topic equivalent to the backend audit/task topic can be created deterministically.
  - A smoke assertion proves at least one backend producer or consumer path against Redpanda.
  - `DONT_USE_KAFKA=true` remains available for narrow MQTT-only tests.
- **Notes:** this is a local validation choice only. Production Kafka/EKS strategy still needs a separate decision record.
- **Verification 2026-04-29:** added Redpanda `v24.3.6` to namespace `backend-deps-local`, exposed it through service DNS `redpanda.backend-deps-local.svc.cluster.local:9092`, and added localhost port-forward `19092`. `./scripts/redpanda-smoke-test.sh` passed by creating a unique topic, producing one message, and consuming it back with `rpk`. Added `manifests/backend-kafka`, `scripts/backend-kafka-up.sh`, and `scripts/backend-redpanda-smoke-test.sh`; the backend-facing smoke test deployed `sensor-alarm-backend` with Kafka enabled, produced a message to `logs-local`, and verified the backend consumer persisted it into local MongoDB `tbl_logs`.

### LK8S-018 - Product namespace map

- **Status:** `Done`
- **Priority:** P1
- **Purpose:** make the local cluster mirror product component boundaries so EKS migration work can be reasoned about service by service.
- **Recommended action:** document and scaffold namespaces as components become active:
  - `mqtt-local` for EMQX
  - `backend-local` for `sensor-alarm-backend`
  - `auth-local` for `sso-provider`
  - `jobs-local` for cron/worker execution
  - `backend-deps-local` for MySQL, MongoDB, Redis, and Redpanda local fixtures
  - `simulator-local` for simulator/test jobs if moved into Kubernetes
  - `frontend-local` for optional Angular/Nginx local portal validation
- **Acceptance criteria:**
  - Namespace ownership and service DNS are documented.
  - Each namespace can be applied idempotently.
  - Local-only dependency namespaces are clearly marked as non-production architecture.
- **Verification 2026-04-29:** added `manifests/product-namespaces`, `docs/product-namespace-map.md`, `scripts/namespaces-up.sh`, and `scripts/namespaces-status.sh`. `./scripts/namespaces-up.sh` created or configured `mqtt-local`, `backend-local`, `backend-deps-local`, `auth-local`, `jobs-local`, `simulator-local`, and `frontend-local`; `./scripts/namespaces-status.sh` confirmed all seven namespaces are `Active`. `./scripts/status.sh` now prints the local product namespace map before resource status.

### LK8S-019 - SSO provider local runtime

- **Status:** `Done`
- **Priority:** P2
- **Purpose:** bring `sso-provider` into the local product validation stack.
- **Recommended action:** inspect `code/sso-provider` runtime dependencies, add a local env fixture, then run it in namespace `auth-local`.
- **Acceptance criteria:**
  - SSO image builds locally.
  - SSO Deployment runs with local-only credentials and dependencies.
  - A readiness or token-flow smoke check can run without AWS or production secrets.
  - Backend/API auth integration points are documented for later end-to-end tests.
- **Verification 2026-04-29:** added a local-only SSO image path, env fixture, Kubernetes ConfigMap/Deployment/Service in namespace `auth-local`, and `sso-image-build`, `sso-up`, `sso-status`, and `sso-smoke-test` scripts. `sso-provider` now skips AWS Secrets Manager in `NODE_ENV=local` and reuses the development JWK set for local signing metadata. The local image uses Node 18 for `oidc-provider@7` runtime compatibility. `./scripts/sso-smoke-test.sh` built and loaded `sensorsyn/sso-provider:local`, deployed one SSO replica, confirmed MySQL connectivity, port-forwarded `svc/sso-provider` to `localhost:4100`, and verified the OIDC discovery document for issuer `http://sso-provider.auth-local.svc.cluster.local:4100`.

### LK8S-020 - Jobs and cron local model

- **Status:** `Done`
- **Priority:** P2
- **Purpose:** move scheduled/background behavior into Kubernetes primitives for local validation.
- **Recommended action:** inventory current cron EC2 jobs and backend scheduled tasks, then model safe local versions as `CronJob`s or worker Deployments in `jobs-local`.
- **Acceptance criteria:**
  - Current job list is documented with source repo/file references.
  - Jobs that require external SaaS calls are disabled, faked, or explicitly skipped locally.
  - At least one safe local job can run on demand and report success/failure.
- **Verification 2026-04-29:** added `code/infra/k8s/docs/jobs-local-model.md` with the current cron inventory covering the prod cron EC2 instance, backend HTTP cron endpoints, Agenda scheduled jobs, communication queues, and property-manager sync paths. Added a suspended `local-stack-smoke` CronJob in namespace `jobs-local`, plus `jobs-up`, `jobs-status`, and `jobs-smoke-test` scripts. The first executable job is intentionally not a business cron; it runs on demand and validates local Kubernetes job mechanics plus in-cluster reachability for EMQX, backend, MySQL, MongoDB, Redis, Redpanda, and SSO, then checks SSO OIDC discovery. `./scripts/jobs-smoke-test.sh` passed by creating Job `local-stack-smoke-1777470070` from the suspended CronJob and completing all reachability/discovery assertions.

### LK8S-021 - Simulator Kubernetes job

- **Status:** `Done`
- **Priority:** P2
- **Purpose:** let the MQTT simulator and scenario runner execute inside the cluster for fully local validation.
- **Recommended action:** package `mqtt-prototype` or the scenario runner as a Kubernetes `Job` in `simulator-local` after the backend app runs in-cluster.
- **Acceptance criteria:**
  - Simulator/test Job connects to EMQX through service DNS.
  - Scenario output is visible in pod logs.
  - The job exits non-zero on failed assertions.
  - Host-run simulator remains available for fast development.
- **Verification 2026-04-30:** added local simulator images, a suspended `mqtt-simulator-scenarios` CronJob in namespace `simulator-local`, and `simulator-images-build`, `simulator-job-up`, `simulator-job-status`, and `simulator-job-smoke-test` scripts. The Kubernetes Job runs the .NET simulator and Node scenario runner as two containers sharing an `emptyDir` signal volume; the runner executes all MQTT scenarios against `emqx.mqtt-local.svc.cluster.local:18756`, then signals the simulator to stop. `./scripts/simulator-job-smoke-test.sh` passed by creating Job `mqtt-simulator-scenarios-1777496278`, running `VERIFY`, `ALARMS`, `ADD`, `BATTERY`, `TEST`, `REMOVE`, `ALERT`, `TAMPERED`, `DISCONNECT`, `RECONNECT`, and `LOW_BATTERY` scenarios in-cluster, and completing successfully. The host-run `./scripts/test.sh` path remains unchanged.

### LK8S-022 - Frontend local validation option

- **Status:** `Done`
- **Priority:** P3
- **Purpose:** decide whether `sensor-angular` should be part of the local Kubernetes product loop.
- **Recommended action:** keep S3/CloudFront as the likely production delivery model, but optionally add an Nginx-served local portal image for API/auth smoke flows.
- **Acceptance criteria:**
  - Decision is documented: host-run Angular, Nginx container in `frontend-local`, or defer.
  - If containerized, local portal uses local API/auth endpoints only.
  - No production CloudFront/S3 configuration is required for local validation.
- **Verification 2026-04-30:** documented the local frontend decision in `code/infra/k8s/docs/frontend-local-model.md`: production stays on S3/CloudFront-style static delivery, while local validation uses an Nginx-served Angular bundle in namespace `frontend-local`. Added a local Angular environment fixture, Nginx config, local image path, frontend manifests, and `frontend-image-build`, `frontend-up`, `frontend-status`, and `frontend-smoke-test` scripts. The build script calls the local Angular CLI directly so it does not trigger the app repo's `prebuild` version bump. `./scripts/frontend-smoke-test.sh` passed after installing `../../sensor-angular` dependencies: it deployed backend, SSO, frontend, port-forwarded `svc/sensor-angular` to `localhost:14200`, verified the SPA shell, and verified the Nginx `/api/` proxy reaches the in-cluster backend service. Npm reported 68 audit findings in the existing Angular dependency tree; dependency remediation is intentionally separate from this local Kubernetes harness item.

### LK8S-023 - Full local stack command

- **Status:** `Done`
- **Priority:** P1
- **Purpose:** provide the easy local validation loop for the product-shaped stack.
- **Recommended action:** add one wrapper that builds/deploys the selected local components and runs the integration suite.
- **Acceptance criteria:**
  - One command validates EMQX, backend dependencies, backend app, Redpanda when enabled, and MQTT/backend assertions.
  - Optional components such as SSO, jobs, simulator Job, and frontend can be enabled by flags or separate targets.
  - The command prints a short component status summary before running assertions.
  - The command requires no AWS credentials and uses no production secrets.
- **Verification 2026-04-30:** added `full-stack-up`, `full-stack-status`, and `full-stack-smoke-test` scripts plus Makefile targets and README usage. The default smoke command renders manifests, applies the stack, prints status across `mqtt-local`, `backend-deps-local`, `backend-local`, `auth-local`, `jobs-local`, `simulator-local`, and `frontend-local`, then runs backend MQTT, backend Redpanda, SSO discovery, jobs reachability, in-cluster simulator, and frontend SPA/proxy assertions. Added env flags to skip optional heavy components during iteration: `SENSORSYN_FULL_STACK_SSO=0`, `SENSORSYN_FULL_STACK_JOBS=0`, `SENSORSYN_FULL_STACK_SIMULATOR=0`, `SENSORSYN_FULL_STACK_FRONTEND=0`, and `SENSORSYN_FULL_STACK_BACKEND_REDPANDA=0`. `./scripts/full-stack-smoke-test.sh` passed end to end. During hardening, `backend-smoke-test.sh` gained configurable `SENSORSYN_BACKEND_READY_TIMEOUT`, and `run-mqtt-scenarios.mjs` now drains/ignores stale MQTT responses so repeated in-cluster simulator Jobs are deterministic.

### LK8S-024 - Local validation documentation refresh

- **Status:** `Proposed`
- **Priority:** P2
- **Purpose:** keep the repo usable for another developer who wants to run the stack locally.
- **Recommended action:** update `code/infra/k8s/README.md` after backend/Redpanda commands exist.
- **Acceptance criteria:**
  - README has a short quick-start for broker-only, backend smoke, and full local stack modes.
  - Troubleshooting covers Docker/kind readiness, image build failures, pod logs, and fixture reset.
  - The Redpanda local role is documented as Kafka-compatible validation, not a production architecture decision.

### LK8S-025 - SSO dependency hygiene

- **Status:** `Proposed`
- **Priority:** P3
- **Purpose:** clean up dependency warnings observed while running `sso-provider` locally.
- **Recommended action:** review and plan upgrades for AWS SDK v2 end-of-support, `oidc-provider@7` support limits, and npm audit findings without destabilizing the local SSO smoke path.
- **Acceptance criteria:**
  - Current warning list is documented with package versions and runtime impact.
  - Upgrade path is proposed separately from the Kubernetes runtime harness.
  - SSO local smoke still passes after any dependency changes.

### LK8S-013 - CI candidate

- **Status:** `Deferred`
- **Priority:** P2
- **Purpose:** make the local MQTT/backend loop eligible for CI once the full local stack is stable.
- **Recommended action:** defer until local backend integration works reliably on a developer machine.
- **Acceptance criteria:**
  - CI can create the kind cluster.
  - CI can deploy EMQX.
  - CI can run the simulator/scenario tests.
  - CI can run any required local backing services.
  - CI always tears down cluster and services.
- **Notes:** CI should use local test credentials only and should not need AWS access.

## V1 Acceptance Criteria

The first local validation milestone is done:

- A local `kind` cluster runs EMQX.
- EMQX is reachable from the host on `localhost:18756`.
- `code/mqtt-prototype` can run outside the cluster against that broker.
- At least `VERIFY`, `ALARMS`, and one event scenario run non-interactively.
- Scenario failures return non-zero exit codes with useful output.
- The entire flow uses local test credentials and no AWS calls.

## Local Product Acceptance Criteria

The next local-product milestone is done when:

- A single command can build and deploy the selected local stack into `kind`.
- EMQX, backend API, MySQL, MongoDB, Redis, and Redpanda run in Kubernetes namespaces.
- The backend app runs in `backend-local` and uses Kubernetes service DNS for local dependencies.
- Redpanda is available as the local Kafka-compatible broker, while `DONT_USE_KAFKA=true` remains available for narrow tests.
- The smoke suite proves MQTT delivery, backend subscriber behavior, DB state changes, and at least one Kafka-compatible producer or consumer path.
- Optional components such as SSO, jobs, simulator Job, and frontend can be added without making the core smoke loop hard to run.
- The validation loop requires no AWS credentials, no production secrets, and no production data.

## Open Questions

- Should the simulator scenario runner live inside `code/mqtt-prototype` or as a separate test-driver project?
- Should the first EMQX deployment use raw manifests or Helm once `helm` is installed?
- Should local EMQX use version `5.8.x` for production parity or latest for tooling convenience?
- Should the simulator eventually support multiple controllers concurrently for shared-subscription and load tests?
- Which backend producer or consumer path should become the first Redpanda assertion?
