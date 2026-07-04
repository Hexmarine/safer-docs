# Haven mass property deactivation via CSV import reconcile — incident + PITR recovery (2026-06-04 → 06-10)

> Archived 2026-07-03 from the Claude memory file `haven-mass-deactivation-2026-06.md` during a memory compaction.
> Point-in-time record — file:line references and live-state claims are as of the dates in the body.


**Incident (discovered 2026-06-10, investigating "missing" MOOR322/205):** the Sensor CSV property import treats the uploaded file as a FULL portfolio snapshot — after import it reconciles and **deactivates every existing agency property not present in the file** (status → `11` DEACTIVATED, `PROPERTY_STATUS` in `constants/app.ts`; flow: `properties.entity.ts` missing-properties block → `jobs.entity.deactivateProperty` in `jobs.entity.ts` ~10890; with agency `autoDeactivation` ON, ACTIVE ones go to `PropertyReviews` first, a sweep (`properties.entity.ts` ~27869) deactivates them ~2 days later).

Our [[haven-property-import-2026-06-05]] uploads from login 59256 ([[haven-ops-login-59256]]) on 2026-06-04 (file ids 20920 errored-all-23, 20921 success; `tbl_property_files.inactiveCount` 278/900) deactivated the ENTIRE pre-existing Haven (agency 37413) portfolio:
- 278 non-ACTIVE properties immediately on 06-04
- 900 ACTIVE properties via the review sweep at **2026-06-06 23:00:05** (single bulk UPDATE)

**Collateral:** 242 open jobs CLOSED with rejectReason "Property no longer in Sensor Global" (237 on 06-04, 5 on 06-06); S138 emails to contractors/agency attempted (likely SES-throttled — both file rows stuck status 0, see [[csv-import-email-throttle-aborts-finalize]]); SIM deactivation attempted for controller SIMs but `tbl_alarms.simStatus` still shows 901 'active' + 1 '403' → appears NOT to have taken effect (verify with Omondo).

**Recovery:** `tbl_properties.previousStatus` preserved exactly (900×1 ACTIVE, 228×8 ACCEPTED, 49×4 NEW, 1×10 DISCONNECTED) → restorable by setting status=previousStatus; closed jobs need separate restore (identify via rejectReason + completionDate 06-04/06-06).

**Side answer:** MOOR322/205 exists as GUID/title `MOOR322/205Z` (property id 16735, trailing Z from the source data at the 2025-08-18 import; MOOR322/102Z likewise). Deactivated rows are hidden in portal lists — that's why "can't find it".

**Rule going forward:** NEVER upload a partial CSV via the property-import for an agency with existing properties unless the reconcile/auto-deactivation behaviour is accounted for — include the full portfolio or use a different path. Guard scripts + runbook now exist: [[property-import-safety-tooling]].

**RECOVERY (executed 2026-06-10, all assertions green):**
- Source of truth: two RDS PITR clones of sensor-prod (A @ 2026-06-04T22:20Z pre-wave-1, B @ 2026-06-06T22:55Z pre-sweep; both deleted after extraction). Extracts + live pre-images + `generate-restore.py` + the applied SQL all in `backups/incident-20260604/`.
- Restored: 1178 properties (status=previousStatus, previousStatus=NULL; per-row cross-check vs clone A: 0 mismatches); 242 jobs to exact prior state (230×ACCEPTED, 10×REJECTED, 2×PENDING incl. job 6384; completionDate/rejectReason restored); 2118 alarms connectedStatus (satellites 0→1; controllers were '0' pre-incident too — hub rows don't use connectedStatus). 1800 tbl_property_review rows neutralized (isCurrentImport=0), files 20920/20921 unstuck (status 0→'1').
- SIM scare resolved: all 902 hub SIMs fine — `lastConnectionTestDate` (stamped by mqtt subscriber.ts:719 → checkHubStatusAndUpdate on inbound hub traffic) shows ALL 902 hubs talked on 2026-06-10; KORE deactivation calls had no fleet effect.
- The 23 new properties keep status 4 + previousStatus 4 (import wrote that itself; pre-dates restore).
- Pending: T+48h re-check (after a review-sweep cron pass) that Haven status-11 count is still 0 and no new tbl_property_review rows; portal eyeball by user.
