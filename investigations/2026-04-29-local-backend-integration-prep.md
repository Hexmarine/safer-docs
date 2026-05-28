# Local Backend Integration Prep

- Date: 2026-04-29
- Scope: local-only discovery for running `code/sensor-alarm-backend` against the local kind/EMQX MQTT validation loop
- Related backlog: `docs/local-k8s-validation-backlog.md`, `LK8S-009`
- Safety: no AWS calls, no production secrets, no remote infrastructure changes

## Summary

The backend can be pulled into the local validation loop, but it should start as a local host process rather than a Kubernetes workload. The service has broad startup dependencies and a few production-shaped assumptions that should be isolated before containerizing it.

Recommended V2 shape:

1. Keep EMQX in local kind using `code/infra/k8s`.
2. Run `sensor-alarm-backend` on the host with `NODE_ENV=local`.
3. Run local MySQL, MongoDB, and Redis fixtures.
4. Set `DONT_USE_KAFKA=true` for the first backend loop.
5. Prove one backend-visible MQTT assertion before broadening services or moving the backend into Kubernetes.

## Backend Entrypoint

Repo: `code/sensor-alarm-backend`

Relevant package scripts:

| Script | Meaning |
|---|---|
| `npm run build` | TypeScript compile via `tsc` |
| `npm run start` | `tsc && ts-node tools/copyAssets && node ./build/index.js` |
| `npm run dev` | nodemon watches `src` and runs root `index.ts` |
| `npm run test-server` | builds and starts `build/test-server.js` with dotenv |

Root `index.ts` behavior:

- Always loads dotenv.
- If `NODE_ENV=local`, it starts `src.App` directly.
- For non-local environments, it calls Secrets Manager through `getSecretByARN()` before starting.

Local backend work should therefore use `NODE_ENV=local` to avoid Secrets Manager.

## Startup Dependencies

`src/index.ts` starts the HTTP server and then imports:

- `./services/mqtt/subscriber`
- `./services/kafka/producer`
- `./services/kafka/consumer`
- `./services/socket/main.socket`
- `./services/socket/request.socket`

`BootClass.Boot()` always attempts:

- MySQL connection via Sequelize.
- MongoDB archived connection.
- MongoDB main connection.

Redis is also started at module import through `src/services/redis/main.redis.ts`, and Socket.IO uses Redis adapter clients from `src/services/socket/main.socket.ts`.

## Local Dependency Classification

| Dependency | Local classification | Evidence |
|---|---|---|
| MySQL | Required for first backend assertion | `BootClass.connectMysqlDB()` authenticates and syncs models; MQTT handler writes through `alarmEntity` / Sequelize |
| MongoDB main | Required for startup; likely required for logs | `BootClass.connectMongoDB()` creates main connection; MQTT handler uses `AlarmLogsModel` and `logsEntity` |
| MongoDB archive | Required by current startup code | `BootClass.connectMongoDB()` creates archive connection unconditionally |
| Redis | Required unless code is gated | Redis client connects on import; Socket.IO adapter connects pub/sub clients |
| Kafka | Bypassable for first local loop | producer and consumer both check `DONT_USE_KAFKA` before connecting |
| Secrets Manager | Bypassable locally | root `index.ts` skips `getSecretByARN()` only when `NODE_ENV=local` |
| External SaaS APIs | Avoid for first assertion | SendGrid, Twilio, Odoo, PropertyMe, SSO, KORE, LaunchDarkly paths exist but should not be part of the first local MQTT assertion |

## Local MQTT Settings

Backend MQTT config comes from `src/config/app.ts`:

- `MQTT_HOST`
- `MQTT_USERNAME`
- `MQTT_PASSWORD`
- `MQTT_PORT`

The subscriber connects with:

- `mqtt.connect(CONFIG.APP.MQTT.MQTT_HOST, { username, password, port, reconnectPeriod: 1000 })`

It subscribes to:

- `$share/sensorGroup/sg/sas/resp/+`

For the current local EMQX harness, use:

```env
MQTT_HOST=mqtt://127.0.0.1
MQTT_USERNAME=
MQTT_PASSWORD=
MQTT_PORT=18756
```

The existing local harness exposes EMQX to the host through `./scripts/port-forward.sh`.

## Health Endpoint

The monitoring route is:

- `GET /api/v1/monitoring/health-check`

It is mounted under `BASE_ROUTES.MONITORING` and protected by `AuthenticateIPs`.

The handler performs real checks:

- queries stalled hubs from MySQL
- queries invalid invoices from MySQL
- pings Redis

This is useful after local fixtures exist, but it is not a simple process-readiness endpoint. A future local runtime slice should either seed the required `api_key` / whitelist rows or add a local-only readiness endpoint.

## MQTT Handler Observations

The backend subscriber handles responses from `sg/sas/resp/{controllerSerial}`.

Important branch behavior:

- `LOW_BATTERY` returns immediately today. It is a known backend behavior gap, not a good first green integration assertion.
- `VERIFY`, `ADD`, `REMOVE`, `TEST`, `BATTERY`, `ALARMS`, `ALERT`, `TAMPERED`, `DISCONNECT`, and `RECONNECT` all flow into MySQL/Mongo and/or notification helpers.
- Kafka is not on the MQTT event path; it remains audit/async infrastructure and can be bypassed in first local tests.

Recommended first backend-visible assertion:

1. Seed local MySQL with one active controller row and one active alarm row in `tbl_alarms`.
2. Run backend with local EMQX, MySQL, MongoDB, Redis, and `DONT_USE_KAFKA=true`.
3. Publish a `DISCONNECT` event to `sg/sas/resp/{controllerSerial}`.
4. Assert the child alarm row for `indexNumber=0` changes `connectedStatus` to `0`.

This avoids the current `LOW_BATTERY` early return and avoids notification success as the first assertion.

## Local Env Needs

`.env.example` is incomplete for the local backend loop: it includes MySQL and Redis examples, but it does not list all MQTT, Mongo, Kafka, Sentry, or local-bypass values needed by the observed code path.

Minimum local env fixture should include:

```env
NODE_ENV=local
PORT=4000
DOMAIN_NAME=localhost

DB_MYSQL_USERNAME=root
DB_MYSQL_PASSWPRD=root
DB_MYSQL_DATABASE=smoke_alarm_local
DB_MYSQL_HOST=127.0.0.1
DB_MYSQL_PORT=3306
DB_MYSQL_DIALECT=mysql

MONGO_DB_URL=mongodb://127.0.0.1:27017/smoke_alarm_local
MONGO_DB_ARCHIVE_URL=mongodb://127.0.0.1:27017/smoke_alarm_archive_local
MONGO_DB_DATABASE=smoke_alarm_local

REDIS_HOST=127.0.0.1
REDIS_PORT=6379
REDIS_PASS=

MQTT_HOST=mqtt://127.0.0.1
MQTT_USERNAME=
MQTT_PASSWORD=
MQTT_PORT=18756

DONT_USE_KAFKA=true
KAFKA_CLIENT_ID=local-smoke-api
KAFKA_HOST=127.0.0.1:9092
KAFKA_GROUP_ID=local-smoke-api

ADMIN_EMAIL=local-admin@example.test
ADMIN_PASS=local-password
SALT=local-salt
JWT_CERT=local-jwt-cert
JWT_ALGO=HS256
ENABLE_CONSOLE_LOG=true
```

Additional placeholders may be required as route/controller modules load, but they should stay local fake values.

## Proposed Next Steps

1. Add local backing-service automation for MySQL, MongoDB, and Redis.
2. Add a local backend env sample without real secrets.
3. Add a seed script for one controller and one alarm row.
4. Start backend with `NODE_ENV=local` and `DONT_USE_KAFKA=true`.
5. Add a backend integration test that publishes `DISCONNECT` and asserts MySQL state.
6. Only after this works locally, decide whether to move the backend process into kind or leave it host-run for local development.

