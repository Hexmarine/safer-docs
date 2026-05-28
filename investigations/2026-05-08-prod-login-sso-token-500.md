# Production Login SSO Token 500

- Date: 2026-05-08
- Scope: Production portal login after API/SSO ASG capacity restoration
- Related systems: `admin.sensorglobal.com`, `api.sensorglobal.com`, `auth.sensorglobal.com`, `SmokeAPI-LB`, `sso-api-lb`, `CodeDeploy_sso-api-dg_d-QKLSZP50D`

## Objective

Identify why portal login still fails after production API and SSO target groups were restored to healthy capacity.

## Sources Used

- Repository files:
  - `code/sensor-angular/src/app/modules/auth/login/service/login.service.ts`
  - `code/sensor-angular/src/app/modules/auth/login/view/login.component.ts`
  - `code/sensor-angular/src/app/constant/urls.ts`
  - `code/sensor-alarm-backend/src/routes/users/v1/sso.routes.ts`
  - `code/sensor-alarm-backend/src/controllers/sso.controller.ts`
  - `code/sensor-alarm-backend/src/entities/sso.entity.ts`
  - `code/sso-provider/src/actions/grants/password.ts`
  - `code/sso-provider/src/services/account.service.ts`
- AWS read-only evidence:
  - ALB target health and ALB access logs for API and SSO
  - CloudWatch logs: `sso-prod-api-logs`, `sso-prod-api-error`, `smoke-prod-api-logs`, `smoke-prod-api-error-logs`
  - Auto Scaling group activity for `CodeDeploy_sso-api-dg_d-QKLSZP50D`
  - CodeDeploy metadata for `d-1L0UPX5BI`

## Findings

- Public infrastructure recovery is complete for the original outage:
  - `SmokeAPI` has healthy API targets.
  - `sso-api-tg` has a healthy SSO target.
  - `https://auth.sensorglobal.com/.well-known/openid-configuration` returns `200`.
- The frontend login flow posts to `https://api.sensorglobal.com/api/v1/users/sso/getTokenViaPassword`.
- API ALB access logs show browser login POSTs reaching API targets, but returning target `400`.
- SSO ALB access logs show matching backend calls from API to `POST https://auth.sensorglobal.com/token`.
- Those SSO token requests return target `500`.
- SSO application logs show the password-grant path reaching MySQL admin lookup and terms lookup, but no session insert or token-success evidence in the inspected window.
- The current SSO instance `i-01606ffe2138c0c8e` was launched at `2026-05-08 07:36:25` Sydney from `prod-sso-api-lt` version `35`.
- The previous SSO instance `i-0d120e1ad6a527ea0` was terminated during the scale-to-zero event on `2026-05-07`.
- CodeDeploy redeployed SSO deployment `d-1L0UPX5BI` from S3 artifact `sso-api/BuildArtif/TpCw7L7` and marked it `Succeeded`.
- SSM inspection of the current SSO host showed the real process is managed by root PM2:
  - Process: `node /home/ec2-user/sso-api/build/src/index.js`
  - PM2 home: `/root/.pm2`
  - Port: `4100`
  - Local logs: `/var/log/pm2/sso-api-dev-out.log`, `/var/log/pm2/sso-api-dev-error.log`
- A fresh marker at `2026-05-07T22:13:27+00:00` showed:
  - SSO starts successfully and logs `Connection has been established successfully` for MySQL.
  - The login attempt reaches SSO and looks up `peresada@gmail.com` with `userType = 8`.
  - The flow reaches the terms-version lookup, which means SSO got past the initial user lookup and password hash comparison for that attempt.
  - No matching `tbl_sessions` insert appears for that user after the marker, so the failure is later than password validation and earlier than final token/session success.
  - No fresh stack trace appeared in the tailed stderr after the marker.
- The process initial environment check via `/proc/2258/environ` did not show the expected runtime variables, but that is not conclusive because this service loads `.env` and then mutates `process.env` from AWS Secrets Manager after startup.
- The SSO error page renderer has a masking bug: `renderError(ctx, out, error)` passes `out` directly to `error.ejs`, while `error.ejs` references `error` and `error_description` as required variables. If `out` lacks those keys, the original auth failure can be replaced by `ReferenceError: error is not defined`.
- A root-level runtime probe on the SSO instance confirmed:
  - `.env` contains bootstrap keys for `NODE_ENV` and `SECRET_NAME`.
  - Secrets Manager loads a MongoDB URI at runtime.
  - MongoDB connects successfully from the SSO host.
  - `CLIENT_SUFIX` is present.
  - The `sensor_api_production_client` Mongo client exists with `authorization_code`, `password`, `refresh_token`, and `client_credentials` grants.
  - The `sensor_api_production_client` scope includes `openid`, `offline_access`, and `api:read`.
  - The `safer_ops_client` exists but only has `authorization_code` and `refresh_token`, which is expected for the Safer Ops browser login path and not relevant to admin portal password login.
- A direct local `POST http://127.0.0.1:4100/token` probe with `sensor_api_production_client`, `grant_type=password`, `compareType=auth`, `userType=8`, and a user-supplied password returned:
  - HTTP `400`
  - `error = UNAUTHORIZED`
  - `error_description = Invalid Credentials`
  - No access token, ID token, or refresh token
- This means the local SSO token endpoint is responding normally rather than failing with an infrastructure/runtime `500`. The remaining failure is likely user lookup, password hash mismatch, user type mismatch, or mismatched `SALT` between services.
- A follow-up password-match diagnostic for `peresada@gmail.com` with `userType=8` showed:
  - User row exists.
  - User status is active.
  - 2FA is disabled.
  - Runtime `SALT` is present.
  - The supplied password matches the stored hash.
- A repeated local `/token` probe after confirming the same env values returned:
  - HTTP `500`
  - `error = server_error`
  - `error_description = oops! something went wrong`
  - No access token, ID token, or refresh token
- Current narrowed conclusion: credentials are valid, but SSO fails during token issuance after password validation. The most likely region is OIDC grant/session/access-token/refresh-token persistence, ID token generation, or cookie/session finalization.
- Mongo inspection of the `basemodels` collection after failed token requests showed repeated partial OIDC writes for the same user:
  - `Grant`
  - `Session`
  - `AccessToken`
  - No `RefreshToken`
- Example local times:
  - `2026-05-08 08:43:58 AEST`
  - `2026-05-08 08:37:52 AEST`
  - `2026-05-08 08:13:36 AEST`
- The `basemodels` collection has a unique `payload.grantId_1` index with a partial filter covering `AccessToken`, `AuthorizationCode`, `RefreshToken`, `DeviceCode`, and `BackchannelAuthenticationRequest`.
- The SSO password grant saves `AccessToken` and then, if refresh tokens are enabled, saves `RefreshToken`. Both token records share the same `grantId`. The unique `payload.grantId` index therefore allows the `AccessToken` insert and rejects the following `RefreshToken` insert.
- Current root cause: invalid unique MongoDB index design on `basemodels.payload.grantId`. This turns otherwise valid password logins into generic SSO token `500` responses whenever a refresh token is issued.
- The same index had been dropped during the May 1 MongoDB custody cutover incident, but it reappeared by May 8. The deployed SSO schema still declares the `payload.grantId` index as `unique`, so fresh SSO starts can recreate the incompatible index.

## Recovery Applied

- The user/operator dropped the `payload.grantId_1` index from `sensorproddb.basemodels` during this investigation.
- Before the change, `basemodels` indexes included `payload.grantId_1`.
- After the change, `basemodels` indexes no longer included `payload.grantId_1`.
- A repeated local `POST http://127.0.0.1:4100/token` probe returned:
  - HTTP `200`
  - `access_token_len = 43`
  - `id_token_len = 1130`
  - `refresh_token_len = 43`
  - `userId = 59117`
  - `userType = 8`

## Risks Or Constraints

- Codex actions stayed read-only for AWS and MongoDB. User/operator performed the SSM diagnostics and MongoDB index drop externally.
- Launch template user data was not read because it may expose runtime secrets.
- SSO token endpoint errors are not clearly emitted to the configured CloudWatch error log or PM2 stderr, so ALB access logs and local SQL progress logs are currently the strongest evidence for where the token flow stops.
- The deployed SSO code still contains the bad unique-index declaration, so the problem can recur after future SSO restarts or deployments until the code is fixed and deployed.

## Recommended Next Steps

- Retry browser login through `admin.sensorglobal.com` and confirm the API login POST returns `200`.
- Check SSO/API logs after the browser retry to confirm `tbl_sessions` insert appears and no new SSO `/token` `500` appears.
- Durable code fix: remove `unique: true` from the `payload.grantId` index in `code/sso-provider/src/db/mongodb/models/BaseModel.ts`, deploy SSO, and ensure no startup path recreates the bad index.
- Consider creating a startup or health diagnostic that checks `basemodels` does not have the incompatible `payload.grantId_1` unique index.
- Fix the SSO error renderer so the original OAuth/auth error is visible instead of being masked by `ReferenceError: error is not defined`.

## Open Questions

- Was the `payload.grantId_1` index recreated by Mongoose auto-indexing on the fresh SSO instance, or by another maintenance/deploy path?
- Why did the unique `payload.grantId_1` index not surface earlier for successful-looking monitoring requests? Confirm whether those requests issued refresh tokens or used a different client/scope.
- Why are SSO token `500` errors not appearing in `sso-prod-api-error` with useful stack traces?
- Does the CodeDeploy artifact assume state that existed on the terminated old SSO instance but is missing from a fresh launch?
