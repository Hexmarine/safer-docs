# Per-property alerts leak in `/users/alarms/alerts` (`getAlarmAlertsList`)

**Date:** 2026-06-21
**Status:** Root-caused, fixed locally (uncommitted), regression-guard PASS. Pending commit + backend deploy.
**Repo:** `code/sensor-alarm-backend`

## Symptom
The new safer-ops Property drawer's "Open issues now" panel (Issues & events tab)
showed a single property (e.g. 22073, Haven) with ~40 *Disconnected* hubs +
foreign alerts whose serials do **not** belong to that property. The property's
own device roster (Devices tab) was correct (3-10 devices). Cross-checked against
the canonical Sensor audit PDF for prop 22073: none of the ~40 leaked hub serials
(`001313569520…`, `001013202227…`, `003613107819…`, water-leak `…026205`, etc.)
appear anywhere in that property's 70-page history or its compliance certificate
— they are other Haven properties' open issues.

## Root cause
`getAlarmAlertsList(params, sessionData)` (`src/entities/alarms.entity.ts`)
builds a Sequelize query whose only root-level filters are `agencyId` +
`eventStatus`. `tbl_alarm_alerts` has no `propertyId` column — it reaches a
property only via `alarmDetails` (Alarms, joined on `alarmId`) → Properties.

The per-property filter was applied as:
```js
query.include[0].include[0]["where"] = { id: params.propertyId };  // Properties (grandchild)
```
But `include[0]` (Alarms / `alarmDetails`) is declared `required: false` — a LEFT
JOIN. Filtering a grandchild of a LEFT-joined parent does **not** prune the root
`tbl_alarm_alerts` rows (the filter lands in the ON clause), so the query fell
back to its only effective predicate, `agencyId`, and returned **every open
alert in the whole agency**. `findAndCountAll` count was likewise agency-wide.

The agency-wide **live monitor** never tripped this because it calls the route
**without** `propertyId` (LEFT JOIN is correct there). The defect only surfaces
on the per-property path.

## Provenance — pre-safer (the decisive question)
| Date | Repo | Event |
|---|---|---|
| 2024-09-25 | sensor-backend | `getAlarmAlertsList` created with the `required:false` LEFT JOIN — `73b29782c2`, SENS-5621 "created alerts list api with pagination" (Gurpreet Singh) |
| 2024-10-18 | sensor-backend | `propertyId` filter pinned on the grandchild Properties include — `d6ea2985a9` (Pedro Veloso) |
| 2026-05-06 | safer-ops | repo's first commit (safer-ops did not exist before this) |
| 2026-06-19 | safer-ops | property history report — first caller anywhere to pass `propertyId` to `/users/alarms/alerts` (`9cce321`) |

The defect predates safer-ops by ~20 months and predates the safer-ops repo's
existence. safer-ops did not introduce it; it is the first code to exercise the
latent per-property path. Call graph confirmed single-consumer: `getAlarmAlertsList`
→ controller `getAlarmAlerts` → route `GET /users/alarms/alerts` (AdminUserAuth);
no `/admins` variant.

## Fix (Class B, default-preserving)
Filter on the adjacent `Alarms` association (which has `propertyId`, indexed) and
make that join required, only on the propertyId path:
```js
if (params.propertyId) {
  query.include[0]["required"] = true;
  query.include[0]["where"] = { propertyId: params.propertyId };
}
```
Absent `propertyId`, `include[0]` stays `required:false` with no where → agency
-wide live monitor byte-identical. `sensor-regression-guard` verdict: **PASS**
(Type B, bounded). Notes from the guard: count/pagination now correct on the
per-property path; `search`+`propertyId` combined is correct but doubly-scoped
(benign); alerts with an unresolvable `alarmId` are excluded on the per-property
path only (correct — cannot attribute to a property).

## Out of scope / not changed
- The agency-wide live monitor path (no propertyId) — untouched.
- The nested Properties include (still used for title/address attributes + sorts).
- sensor-mcp `getActiveAlerts` also benefits once deployed (same endpoint).

## Deploy
Backend change → needs its own commit + deploy (separate from safer-ops). The
safer-ops drawer consumes the corrected scoping automatically once live; no
safer-ops change required for this fix. Record in `docs/applied-changes.md` once
deployed + confirmed.
