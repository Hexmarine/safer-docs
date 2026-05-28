# SensorSyn API Playground

- Date: 2026-05-02
- Scope: Production SensorSyn HTTP API exploration using local `SENSOR_USER` and `SENSOR_PASS` environment variables
- Related systems: `https://api.sensorglobal.com/api/v1/`, Angular admin/agency/trader portals, backend SSO password flow

## Objective

Establish a safe, repeatable way to make authenticated read-only API calls for system exploration without committing credentials, tokens, or response dumps to the repository.

## Sources Used

- Repository files:
  - `code/sensor-angular/src/app/constant/urls.ts`
  - `code/sensor-angular/src/app/modules/auth/login/service/login.service.ts`
  - `code/sensor-alarm-backend/src/routes/index.ts`
  - `code/sensor-alarm-backend/src/routes/users/v1/sso.routes.ts`
  - `code/sensor-alarm-backend/src/routes/v1/accounts.routes.ts`
  - `code/sensor-alarm-backend/src/routes/v1/properties.routes.ts`
  - `code/sensor-alarm-backend/src/routes/v1/locale.routes.ts`
- Live API checks:
  - `GET /api/v1/monitoring/health-check`
  - `POST /api/v1/users/sso/getTokenViaPassword`
  - `GET /api/v1/admins/profile?validateSSO=true`
  - `GET /api/v1/admins/locale/drop_down`
  - `GET /api/v1/admins/properties/list?page=1&limit=1`

## Findings

The API origin for direct calls is `https://api.sensorglobal.com/api/v1/`. The portal host, for example `https://admin.sensorglobal.com`, serves the Angular app and is not the direct JSON API origin.

The current portal login path uses the SSO password-flow endpoint:

```bash
curl -sS -o /tmp/sensorsyn-login.json \
  -w '%{http_code} %{content_type}\n' \
  -X POST 'https://api.sensorglobal.com/api/v1/users/sso/getTokenViaPassword' \
  -H 'content-type: application/json' \
  --data-raw "{\"email\":\"$SENSOR_USER\",\"password\":\"$SENSOR_PASS\"}"
```

For subsequent admin API calls, use the `id_token` returned by the login response as an `access_token` header value:

```bash
jq -r '.data.id_token // empty' /tmp/sensorsyn-login.json > /tmp/sensorsyn-id-token
chmod 600 /tmp/sensorsyn-id-token

curl -sS 'https://api.sensorglobal.com/api/v1/admins/profile?validateSSO=true' \
  -H "access_token: bearer $(cat /tmp/sensorsyn-id-token)"
```

Verified live behavior:

| Call | Result | Notes |
| --- | --- | --- |
| `GET /monitoring/health-check` without auth | `403` | Public unauthenticated access is blocked. |
| `POST /users/sso/getTokenViaPassword` | `200` | Returned `id_token`, `access_token`, and `refresh_token`. Do not print or commit these. |
| `GET /admins/profile?validateSSO=true` | `200` | Authenticated profile returned user type `8` and role ID `41`. In backend constants, user type `8` is `SUB_ADMIN`. |
| `GET /admins/locale/drop_down` | `200` | Returned locale/testing cadence reference data. |
| `GET /admins/properties/list?page=1&limit=1` | `200` | Returned `count: 18317`, `page: 1`, and one property row. |

Useful read-only probes:

```bash
curl -sS 'https://api.sensorglobal.com/api/v1/admins/locale/drop_down' \
  -H "access_token: bearer $(cat /tmp/sensorsyn-id-token)" |
  jq '{code, message, data}'

curl -sS 'https://api.sensorglobal.com/api/v1/admins/properties/list?page=1&limit=5' \
  -H "access_token: bearer $(cat /tmp/sensorsyn-id-token)" |
  jq '{code, message, count:.data.count, page:.data.page, rows:(.data.data | length)}'
```

## Risks Or Constraints

- These credentials and tokens must remain local. Do not write them to repository files, docs, shell history snippets, PRs, or chat output.
- The login call is not purely read-only: it creates an application session and audit/log side effects. Treat it as the narrow authentication exception needed before read-only exploration.
- Production API exploration should stay on `GET` endpoints unless there is explicit approval for a concrete mutation. Avoid `POST`, `PUT`, `PATCH`, and `DELETE` endpoints after login.
- Some GET endpoints expose personally identifiable or operational data. Prefer aggregate counts, key summaries, and small `limit` values when documenting findings.
- The profile returned a sub-admin role. Role permissions should be checked with real endpoint responses rather than assumed from the role name.

## Cost Optimization Opportunities

None. This investigation was about API access mechanics, not cost.

## Recommended Next Steps

- Build a small local-only helper script only after agreeing on the safe endpoint allowlist.
- Keep live exploration outputs under `/tmp` and summarize only non-sensitive response shapes in durable docs.
- Use `limit=1` or `limit=5` first when probing list endpoints.
- If a request returns `403`, `401`, or a permission error, record the endpoint and role context rather than trying broader or mutating alternatives.

## Open Questions

- Which read-only endpoint groups should be considered safe for routine playground use?
- Should API exploration target production only, or should equivalent QA/sandbox origins be revalidated for lower-risk testing?
- Should a local helper cache only `id_token`, or also support refresh-token renewal?
