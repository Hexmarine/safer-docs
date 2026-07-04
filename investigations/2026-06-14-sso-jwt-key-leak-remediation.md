# Committed prod SSO/JWT signing key — exposure, externalisation, rotation (remediated 2026-06-14)

> Archived 2026-07-03 from the Claude memory file `sensor-prod-jwt-key-in-repo.md` during a memory compaction.
> Point-in-time record — file:line references and live-state claims are as of the dates in the body.


`code/sensor-alarm-backend/src/config/sso.key.ts` hardcodes full RSA **private**
keys (n/e/d/p/q/dp/dq/qi) per environment, including the `production`/`prod`
branch. Committed 2024-03-20 (`b1db140d9`, jeetendra.singh@appinventiv.com,
SENS-4848), never rotated.

**Exactly how the key is used (traced 2026-06-06):** `jwkKey` is imported in ONE
place — `libs/TokenManager.ts`. There `privatePEM` signs and `publicPEM` verifies
(RS256). `generateUserToken` mints 4 OUT-OF-BAND token types: `FORGOT_PASSWORD`
(reset links), `END_USER_LOGIN` (tenant/landlord magic-link login),
`USER_LOGIN`, `FOLDER_VERIFY` (1-day; rest 180-day). `verifyToken` validates them
in the reset/validate/magic-link flows.

**Scoping (important):** this key is NOT the live API-session key. `AdminUserAuth`
(`middlewares/auth.ts:182`) verifies request bearer tokens with
`getKeyDetails.getPublicKey()` — the **sso-provider** key (issuer localhost:4100,
see [[oidc-expected-issuer-localhost]]), a different key. So repo read alone can't
directly forge an API session token. Practical blast radius via jwkKey:
(1) forge `FORGOT_PASSWORD` → reset ANY account's password → then log in normally
= full account takeover (proven end-to-end this session); (2) forge
`END_USER_LOGIN` → directly impersonate any tenant/landlord. Worth confirming
whether the sso-provider session key is ALSO committed.

Compounding weakness: `resetPassword` / `resetPasswordWithOTP`
(`userAccounts.entity.ts:1201` / `:1366`) verify the JWT signature but DON'T
compare the JWT's embedded reset token to `tbl_admins.token` — they only check
the column is non-empty (and for OTP, `code == mailOTP`).

**Why it matters:** key should move to a secret (Secrets Manager, like SALT) and
be rotated; reset flow should match the stored token. We relied on this to mint
an inbox-free reset for [[haven-ops-login-59256]], but it's a real exposure worth
raising with the team / writing up as an issue.

**UPDATE 2026-06-14 — remediation built + reviewed, Phase-0 deploy planned.**
Confirmed the sso-provider session key IS also committed (byte-identical `vXYg…`
prod modulus in BOTH `backend/src/config/sso.key.ts` and
`sso-provider/src/configs/jwk.key.ts`) — so the leaked key signs LIVE OIDC
sessions too, not just out-of-band reset/magic-link tokens. Full blast radius =
direct session forgery, all tenants.
- Code: externalized to env `SSO_SIGNING_JWKS` (JSON array; [0]=active signer,
  all verify) + provider cookie key to `SSO_COOKIE_KEYS`; verifiers resolve by
  header `kid` (multi-key, rotation-safe). Backend committed `ef229a60e` on
  `fix/remove-leaked-sso`; provider uncommitted on `main`. codex-clean.
- **CORRECTED 2026-06-14 (caused an incident): BOTH prod services use `vXYg`.**
  My first analysis was WRONG. I thought backend used dev-block `sqy3` (because OS
  `NODE_ENV=dev`), but the `sensor-prod` secret sets `NODE_ENV=production` which is
  merged BEFORE the lazy `require` of sso.key → old backend selected its
  `production` block = `vXYg` (same as provider). The "byte-identical production
  modulus in both files" = they SHARE `vXYg` in prod. `sqy3` is dev/local ONLY.
  Backend `TokenManager.verifyToken` (cbpf login `tokenValidation` + password-reset
  + folder-verify, 9 sites in userAccounts.entity.ts) verifies the provider's
  `vXYg` OIDC token → backend key MUST be `vXYg`. Putting `sqy3` broke agent login.
  Backend `SSO_SID` (JWKS lookup fallback → provider key) is separate from backend
  `SSO_SIGNING_JWKS` (TokenManager key) but BOTH point at `vXYg` in prod.
- **Deploy asymmetry:** both run EC2+pm2 via CodeDeploy (k8s = local-dev only);
  secrets merge via `Object.assign(process.env, secret)` (backend=`sensor-prod`,
  provider=`sensor-prod-sso`). Provider fail-fasts if secret missing; backend
  (NODE_ENV=dev) silently loads throwaway dev key → populate secret BEFORE deploy.
- Plan: `~/.claude/plans/wondrous-tinkering-yeti.md` (Phase-0 deploy+rollback +
  graceful-rotation follow-on). Rebuild-trap check passed: grantId index
  non-unique, deps lockfile-pinned. gen-jwks.js at workspace-root scripts/.
- **Step 1 DONE 2026-06-14:** Phase-0 secrets POPULATED (elevated mutation
  window). `sensor-prod-sso` += `SSO_SIGNING_JWKS`(=[vXYg] kid-less)+`SSO_COOKIE_KEYS`(=["subzero"]) → 17 keys; `sensor-prod` += `SSO_SIGNING_JWKS` → 159 keys (FIRST set to [sqy3] = WRONG, broke login; CORRECTED to [vXYg] in the incident below); backend `SSO_SID` untouched. Done via
  `scripts/phase0-populate-secrets.js --apply` (extracts keys from git, no
  values printed; add-only guard = idempotent). Continuity verified: vXYg
  thumbprint == live JWKS kid `2lzYsX9wUIn…`. Logged in docs/applied-changes.md.
- Runtime facts (diagnostic window): backend NODE_ENV=dev (inline), SECRET_NAME=
  sensor-prod, app dir /home/ec2-user/smoke_api, 2 prod-api-server instances
  (i-0db346e3a3050e38f, i-056c83002b94cb04c) + cron i-0392d1fca678f1a21 (no live
  node). Provider NODE_ENV="Prod" in .env (effectively production via secret),
  SECRET_NAME=sensor-prod-sso, /home/ec2-user/sso-api, i-0d6e3a859fd9733b1. Both
  load key modules via lazy `require` AFTER getSecretByARN merge (verified in
  index.ts) — so secret values are seen. AWS CLI 2.31/Py3.14 ssm send-command is
  broken → drive SSM/SecretsManager via boto3/aws-sdk. get-secret-value is
  policy-DENIED (can't read back; rely on add-only guard).
- **Step 3 BACKEND DONE 2026-06-14:** backend PR #6035 merged → CodePipeline
  `Smoke-API` → CodeDeploy `d-WH8O6CVZI` (BLUE/GREEN via ASG) succeeded. Live
  green instances behind ALB tg `SmokeAPI`:4553 = i-0d729d02217754fed +
  i-0ad1e502aec0c31a4 (old blue i-0db…/i-056c… drained → diagnostics must target
  GREEN). Verified live: Application started (secret merged → sqy3 not dev
  fallback), build/src markers present, rsa.key gone, healthy. cron box i-0392…
  is NOT in the deploy group. Backend prod is BLUE/GREEN — don't probe drained
  blue instances.
- **Step 3 PROVIDER DONE 2026-06-14:** sso-provider PR #174 merged → CodePipeline
  `sso-api` → CodeDeploy `d-COWVX7VZI` (BLUE/GREEN, ASG `sso-api-dg`) succeeded.
  Green sso instance = i-0285ac5c6596bd521 (healthy, sso-api-tg:4100); blue
  i-0d6e… drained. **JWKS CONTINUITY HELD: kid still 2lzYsX9wUIn… (vXYg)** — Phase-0
  goal achieved, in-flight tokens unaffected. sso ALB health check is `GET /`
  (doesn't hit /token) so E11000 grantId trap not functionally disproven by health
  (but structurally neutralised: grantId index non-unique).
- **BOTH Phase-0 services now LIVE in prod (2026-06-14).** Externalisation
  complete; leaked key still the active signer (preserved).
- Green sso i-0285… verified: new code, jwks 200, fresh boot, **E11000/grantId
  errors = 0** (rebuild trap did NOT hit).
- **INCIDENT 2026-06-14 (caused by me, fixed): agent login down ~50 min.** The
  `/users/sso/cbpf` login callback → `tokenValidation` → `TokenManager.verifyToken`
  verifies the provider's `vXYg` OIDC token, but I'd populated `sensor-prod`
  `SSO_SIGNING_JWKS` with the wrong key (`sqy3`). So the backend rejected ALL
  provider tokens → `INVALID_LINK "This link has expired"` on every login (and any
  TokenManager-verify flow). I initially MISREAD the cbpf error as "just a stale
  link" — it was a real regression. FIX: `node /tmp/fix-backend-key.js --apply`
  set `sensor-prod.SSO_SIGNING_JWKS=[vXYg]`, then rolling `pm2 restart Smoke_alarm`
  on green i-0d729…/i-0ad1… via SSM. Both rebooted, Application started, healthy.
  Lesson: TokenExpiredError requires a valid sig first — but ALSO verify which KEY
  the verify path actually needs; exercise TokenManager-verify (not just the
  provider JWKS) in Phase-0 checks.
- **PHASE 0 COMPLETE + login CONFIRMED working 2026-06-14** (after the incident
  fix). Both prod services externalised + live on `vXYg`, JWKS continuity held,
  agent login verified by user. Leaked key still the active signer (preserved).
- **ROTATION IN PROGRESS 2026-06-14.** Reload mechanism = `pm2 restart` on green
  instances via SSM (NOT pipeline re-run — avoids provider rebuild-trap risk).
  - **Phase 1 DONE:** new verify-only key kid=`biM4zB_XMt8XRg1l-rW6SWB7Ml4KDgOxU5LIpyy546w`
    appended to BOTH secrets `SSO_SIGNING_JWKS` = `[vXYg(active,[0]), NEW([1])]`;
    all 3 green instances restarted; public JWKS publishes both, vXYg still [0]=signer.
    Script `scripts/rotate-phase1.js`.
  - **Phase 2 DONE:** both secrets reordered → `[NEW, vXYg]`; `sensor-prod.SSO_SID`
    = biM4zB; all 3 green restarted. Public JWKS [0]=biM4zB(signer),[1]=vXYg(verify).
    New tokens now signed by NEW; vXYg still verifies (no forced re-login yet).
    Script `scripts/rotate-phase2.js`.
  - **Phase 3 DONE 2026-06-14 (~22:50 AEST):** vXYg REMOVED from both secrets →
    `[NEW]` only; all 3 green restarted; public JWKS = 1 key (biM4zB), vXYg absent.
    **Leaked key is dead** — can no longer sign/verify. Mobile fleet + pre-Phase-2
    web sessions forced 1× re-login. Script `scripts/rotate-phase3.js`.
- **REMEDIATION COMPLETE 2026-06-14.** Active prod signing key = NEW
  `biM4zB_XMt8XRg1l-rW6SWB7Ml4KDgOxU5LIpyy546w` (generated fresh, never committed,
  in Secrets Manager only). The committed vXYg/sqy3 keys in git are now useless.
- **Cookie key ALSO rotated 2026-06-14:** SSO_COOKIE_KEYS `["subzero"]` → `[NEW
  random]` (Phase A add → Phase B drop subzero), provider-only (sensor-prod-sso).
  Note: each provider pm2 restart = ~few-sec blip on the SINGLE sso instance →
  in-flight login(INVALID_CREDENTIAL)/logout(504) fail transiently then recover.
  For future provider changes, prefer blue/green pipeline to avoid the blip, or
  warn that brief login disruption is expected.
- REMAINING (optional/low-pri): rotate leaked non-prod keys (qa/sandbox/uat);
  optionally scrub git history of dead key blobs. See plan
  `~/.claude/plans/wondrous-tinkering-yeti.md`.
