# Open Questions & Walkthrough Backlog

Working list of questions accumulated during the product/platform walkthrough. Questions resolved by reading code or existing docs are linked to the relevant artefact. Items marked **GAP** are unresolved and need either external access or a conversation with the original product/operations owners — see [`gaps.md`](./gaps.md) for the consolidated access/ownership ask list.

Status legend:
- ✅ resolved (linked artefact answers it)
- 🟡 partial (some context, needs confirmation)
- ❌ GAP (no internal source can answer; needs external input or new access)

---

## Domain & Entity Relations

| # | Question | Status | Reference |
|---|---|---|---|
| 1 | Agent entity relations | ✅ | `diagrams/domain-model.puml` |
| 2 | Contractor entity relations & how they're created | ✅ | `diagrams/domain-model.puml`, walkthrough checkpoint 002 |
| 3 | What is "Staff" (Service Staff role) | ✅ | `diagrams/domain-model.puml` (userType 5, sub-role under Contractor) |
| 4 | Property owners — what they are and their relations | ✅ | `diagrams/domain-model.puml` (Owner package: Landlord, Tenant) |
| 5 | Tenant entity relations & legality | 🟡 | Diagram done; legal/compliance side needs follow-up with **Kristyn** |
| 6 | Lease management end-to-end | 🟡 | `flow-lease-lifecycle.puml` covers lifecycle; operational ownership unclear |

## Property Configuration & Notifications

| # | Question | Status | Reference |
|---|---|---|---|
| 7 | Property `alertConfiguration` — how it's used, can we propagate defaults | ✅ | Defines notification routes/templates per event×recipient×channel; system defaults exist in `DEFAULT_PROPERTY_CONFIGURTION` (`backend/src/constants/app.ts`); per-property overrides tracked in `customalertConfiguration` |
| 8 | Extend notification routing to a specific contact (not just role) | ❌ GAP | Current model is role-based only (agent/agency/tenants/landlord/contractor). Per-contact routing is a feature gap — needs product decision |
| 9 | Property audit history | ❌ GAP | `tbl_property_history` exists; coverage and retention need confirming |

## Operational & Stats

| # | Question | Status | Reference |
|---|---|---|---|
| 10 | Stats on alarms (counts, fire rates, etc.) | ❌ GAP | No dashboard identified; would need direct DB / NewRelic / CloudWatch access |
| 11 | Confirm there is live traffic from hubs | ❌ GAP | Need MQTT broker (EMQX) live inspection access |
| 12 | Error / incident management workflow | ❌ GAP | No internal runbook documented; ask ops owner |

## Environments & Access

| # | Question | Status | Reference |
|---|---|---|---|
| 13 | Can we impersonate as an agency? | ❌ GAP | No "login-as" feature visible in code; need product owner confirmation if it exists out-of-band |
| 14 | Get a sandbox/soapbox environment | ❌ GAP | UAT discovered (`2026-04-16-uat-environment-discovery.md`) but write access / seed data not granted |

## Firmware & Hardware

| # | Question | Status | Reference |
|---|---|---|---|
| 15 | Firmware source code and build process | 🟡 | Confirmed: hubs/sensors & firmware were third-party-developed for SensorGlobal — `infra/company-and-ownership.md`. Source not in `code/` (empty `firmware` repo); vendor identity, repo location & build/OTA pipeline still needed — `gaps-detailed.md` §6 |

---

See [`gaps.md`](./gaps.md) (compact) or [`gaps-detailed.md`](./gaps-detailed.md) (full) for what we need from original product owners to close the ❌ items above.
