# SIM Activation Flow

- Date: 2026-05-02
- Scope: KORE/Twilio Super SIM activation, SaaS controller records, backend status updates, and operational tooling
- Related systems: `code/sim-control`, `sensor-alarm-backend`, KORE Super SIM API, `tbl_alarms`, KORE callbacks

## Objective

Understand how SensorSyn SIMs are activated, which identifiers are used at each step, and where KORE status is synchronized into the application.

## Sources Used

- Repository files:
  - `code/sim-control/README.md`
  - `code/sim-control/Program.cs`
  - `code/sim-control/KoreAPI.cs`
  - `code/sensor-alarm-backend/src/controllers/alarms.controller.ts`
  - `code/sensor-alarm-backend/src/entities/alarms.entity.ts`
  - `code/sensor-alarm-backend/src/routes/v1/alarms.routes.ts`
  - `code/sensor-alarm-backend/src/routes/users/v1/settings.route.ts`
  - `code/sensor-alarm-backend/src/models/mysql/alarms.ts`
  - `docs/investigations/2026-04-12-upstream-tech-docs-review.md`
- Upstream references:
  - `docs/upstream/Technical Documents/SENSORGLOB-KORE Super SIM-110426-072331.pdf`
- AWS commands or consoles: none
- External references: none

## Findings

SensorSyn uses KORE Super SIM, formerly Twilio Super SIM, for hub cellular backhaul. The upstream KORE note says Australian pilot SIMs prefer Telstra first, then Optus and Vodafone, and that a SIM in `ready` transitions automatically to `active` after 250 KB of data, 5 SMS commands, or 3 months.

There are two practical activation layers:

1. **Factory/provisioning activation in KORE.** The `sim-control` .NET tool is the operational bulk tool. It takes a CSV of `controller serial,ICCID`, looks up each SIM in KORE by ICCID, then writes an output CSV of `serialNumber,simId` for SaaS Admin. In this output, `simId` is the KORE/Twilio SIM SID, not the ICCID. The activation command sets KORE `Status=ready`, `UniqueName=<controller serial>`, and `Fleet=<KORE_FLEET_SID>`.
2. **Runtime/property activation in SaaS.** Once a controller is installed against a property, the backend verifies the stored SIM SID against KORE, records the returned KORE status in `tbl_alarms.simStatus`, and, when a property ID is present, calls KORE to move the SIM to `active`.

The identifier distinction matters. The first input file uses ICCIDs because the physical SIM card has that identifier. The SaaS-side `tbl_alarms.simId` is treated as a KORE SIM SID; the current backend rejects SIM IDs that do not start with `HS` before attempting a status change.

`tbl_alarms` stores both the SIM identifier and status:

- `simId`: nullable string, used by current backend code as the KORE SIM SID
- `simStatus`: string defaulting to `inactive`

The accepted status vocabulary in code is:

- `inactive`
- `active`
- `ready`
- `scheduled`
- `404` / not found

### Factory/Batch Provisioning

`code/sim-control` requires:

- `KORE_CLIENT_ID`
- `KORE_CLIENT_SECRET`
- `KORE_FLEET_SID`

The intended factory workflow is:

1. Prepare a CSV with controller serial and physical SIM ICCID.
2. Run `dotnet run activate sims-filename.csv`.
3. For each row, the tool queries KORE `/Sims?Iccid=<iccid>`.
4. If found and not already active, it posts to `/Sims/{sid}` with:
   - `Status=ready`
   - `UniqueName=<controller serial>`
   - `Fleet=<KORE_FLEET_SID>`
5. It writes `output-<input-file>` with `serialNumber,simId`, where `simId` is the returned KORE SID.
6. That output is uploaded into SaaS Admin through the existing controller/SIM upload path.

The tool also has review and process modes:

- `review`: resolves ICCIDs to KORE SID and current KORE status.
- `process`: resolves ICCIDs to KORE SID without changing status.

The same tool has support commands for viewing active fleet SIMs and deactivating active SIMs that are not present in `sims-in-use.csv`. Those commands are operationally risky because they mutate KORE status and should remain human-approved support actions.

### SaaS Upload And Storage

The backend upload path reads CSV rows with `serialNumber` and `simId`. For each valid row it creates or updates a controller record and stores:

- `serialNumber`
- `simId`
- `controller=1`
- `connectedStatus=DISCONNECTED`
- `simStatus=inactive`

This upload step is a database registration step. It does not, by itself, activate the SIM in KORE.

### Property Install Activation

When adding/installing a controller to a property, the backend:

1. Calls `verifySim(simId)` against KORE `/Sims/{sid}`.
2. Rejects the operation if KORE reports not found.
3. Creates or updates the controller row using the KORE-reported `status` as `simStatus`.
4. If `propertyId` is present, calls `changeSimStatus(simId, active)`.
5. Emits audit/log events saying the SIM card was activated because the hub was installed.

This is the main in-app activation path: controller installed on property -> KORE SIM status set to `active`.

### Manual Status Update

The admin route `PUT /api/v1/alarms/update-sim-status` accepts:

- `simId`
- `simStatus` of `active`, `inactive`, or `ready`

It calls KORE first, then updates `tbl_alarms.simStatus` to the status returned by KORE. It also writes an alarm log for `SIM_STATUS_CHANGED`. The route is protected by admin auth and alarm edit permissions.

### Callback And Sync Paths

There are two callback/sync paths:

- `GET /api/v1/alarms/get-sim-status` expects Twilio-style query fields `SimSid`, `SimStatus`, and `AccountSid`, then updates the local `simStatus`.
- `POST /api/v1/users/settings/kore-sim-events` accepts KORE event callbacks and verifies a KORE signature using `KORE_SECRET`, `API_DOMAIN`, and the fixed path `/api/v1/users/settings/kore-sim-events`.

There is also a controller sync path that queries KORE directly and refreshes the local `simStatus`.

### Billing/Subscription Driven Status Changes

Several application flows change SIM status after initial install:

- When an invoice is paid for a suspended property, the backend activates an inactive SIM, marks alarms connected, and sends `VERIFY` over MQTT.
- When a subscription is suspended or a property import/webhook deletes a property, the backend deactivates the SIM and disconnects alarm records.
- Some user/property deletion flows temporarily deactivate and reactivate around controller removal/reset workflows.

## Risks Or Constraints

- **Identifier drift:** Older docs describe `tbl_alarms.simId` as an ICCID, but current runtime code treats it as a KORE SIM SID and requires the `HS...` prefix before status mutation. This is important for imports and support scripts.
- **Activation wording ambiguity:** `dotnet run activate` moves SIMs to KORE `ready`; property installation later moves them to `active`. In KORE billing/lifecycle terms, `active` may also occur automatically after data/SMS/time thresholds.
- **Operational mutation risk:** `sim-control activate`, `ready`, `active`, `inactive`, and `sims deactivate` all mutate KORE state. They should not be run as casual investigation commands.
- **Callback transition complexity:** Backend code still contains a Twilio-era GET callback path and a newer KORE signed POST event path. Both need production verification before any migration or credential rotation.
- **Local DB truth can lag KORE:** `tbl_alarms.simStatus` is synchronized by explicit calls/callbacks, not guaranteed to be a continuously fresh source of truth.

## Cost Optimization Opportunities

SIM deactivation is already partially supported by `sim-control sims deactivate` and by backend subscription/property flows. The safe cost opportunity is an audit-only report first:

- Compare KORE active SIMs with `tbl_alarms` controllers installed on active properties.
- Flag active SIMs with no active installed controller or suspended/deleted properties.
- Review with operations before any deactivation.

No deactivation should be automated until KORE status, SaaS property status, billing status, and support exceptions are reconciled.

## Recommended Next Steps

1. Produce a read-only SIM reconciliation report: KORE status vs `tbl_alarms.simId/simStatus/propertyId/status`.
2. Confirm in production whether KORE callbacks arrive on the signed `/users/settings/kore-sim-events` path, the older `/alarms/get-sim-status` path, or both.
3. Update older migration docs that call `simId` an ICCID, or clarify that physical ICCID is converted to KORE SID before SaaS upload.
4. Document a human-approved runbook for factory SIM provisioning, including sample CSV shape, dry-run/review mode, and approval requirements before status changes.
5. Add guardrails to `sim-control` before future use: dry-run default, explicit `--apply`, safer missing-SIM handling, and clearer output naming.

## Open Questions

- Which KORE account/fleet is currently authoritative for production SIMs?
- Are both Twilio-style and KORE-style callbacks configured in production, or is one legacy/dead code?
- Does SaaS Admin upload currently accept only the `output-*` SID CSV, or do any users still upload physical ICCIDs directly?
- Are there exception SIMs that should remain active without a currently installed property, such as spare/test hubs?
