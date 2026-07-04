# Hub `lowBattery` spuriously set — varchar `batteryStatus` string comparison

**Date:** 2026-06-26
**Status:** Code fix applied (uncommitted, working tree); deploy + data cleanup pending
**Surfaced via:** routine investigation of property `GREE6/403` (id 21931, agency 37413 Haven), hub id 4590 (serial `003512495693C0012406`)

## Summary
`updateAlertOnHub` recomputes a hub's aggregate `lowBattery` flag from its active
children. The low-battery probe compared the **varchar** `batteryStatus` column to the
number `15` via Sequelize `{ [Op.lte]: 15 }`. Because the column/attribute is typed as a
string, Sequelize binds `15` as the **string `'15'`**, emitting SQL
`batteryStatus <= '15'` — a **lexical** comparison in which `'100' <= '15'` is **TRUE**
(`'1'='1'`, then `'0' < '5'`). So full-battery (100%) children matched the "≤15" probe and
the hub's `lowBattery` was set to `1`.

**Blast radius (live, MySQL `tbl_alarms`):** 2,358 active hubs flagged `lowBattery=1`; of
those **2,340 have a numerically healthy battery (>15)** — ~99% of all hub low-battery
flags were spurious. 563 are flagged at exactly `batteryStatus='100'`.

**Impact: display-only.** Nothing reads `WHERE lowBattery=1` to send a comm (verified by
tracing all 25 `lowBattery` references; low-battery emails/SMS fire off the live MQTT
`BATTERY<=15` frame, not this column). So the symptom is ~2,340 misleading operator-portal
"low battery" badges + `readStatus=0` unread markers — not a notification incident.

## Root cause (proven live)
On hub 4590, whose two active children both report `batteryStatus='100'`:

| Predicate | Result |
|-----------|--------|
| `batteryStatus <= 15` (numeric) | `[]` — no match (correct) |
| `batteryStatus <= '15'` (string, = Sequelize-on-varchar) | matches both children (the bug) |
| `CAST(batteryStatus AS UNSIGNED) <= 15` (the fix) | `'100'→0` not low, `'13'→1` low (correct) |

`tbl_alarms.batteryStatus` is `varchar(20)` (`src/models/mysql/alarms.ts`,
`DataTypes.STRING(20)`, `declare batteryStatus: string`).

**Deployed == source:** the live build (`build/src/entities/alarms.entity.js`, v1.1.0,
mtime 2026-06-25 20:02) compares numerically in JS with the same `Op.lte:15` probe — i.e.
the bug is in **current** code, not a stale artifact. The string comparison is introduced by
Sequelize at SQL-generation time and is invisible in both the `.ts` source and the compiled
`.js`.

GREE6/403's hub itself is healthy: live VERIFY → `POWERSTATE 1` (AC), `BATTERY 100`; both
water-leak children 100%. The flag was `0` before the investigation only because
`updateAlertOnHub` had not run on that hub since its children's `batteryStatus` settled to
`'100'`; a battery poll triggered the first recomputation and flipped it.

## Fix (applied, working tree)
`src/entities/alarms.entity.ts`:
- **`updateAlertOnHub`** low-battery probe (~:2644): `batteryStatus: { [Op.lte]: 15 }` →
  `[Op.and]: [ Sequelize.where(Sequelize.literal("CAST(\`batteryStatus\` AS UNSIGNED)"), { [Op.lte]: 15 }) ]`.
  This is the actual fix — it forces a numeric comparison.
- Hardening (behaviour-preserving) `Number(...)` wraps at the JS comparisons:
  `updateAlertOnHub` (~:2647), `verifyAlarm` (~:1887), `updateAlarmUsingIndex` (~:2052).

Test: `test/updateAlertOnHubLowBattery.unit.test.ts` (Mocha/sinon/proxyquire, no DB) locks
the CAST predicate into the probe and asserts a healthy 100% hub yields `lowBattery=0`.
Full unit suite: **79 passing** (run with `--no-experimental-strip-types` on node 24; the
project's canonical node/ts-node loads `.ts` as CommonJS — see note below).

**Regression-guard verdict: SAFE.** CAST = intended C-class fix; the `Number()` wraps are
exact no-ops for real inputs; NULL and `''` behaviour unchanged; genuinely-low `'13'`
preserved. One LOW edge: a non-numeric junk value (e.g. `'abc'`) now matches as low
(`CAST→0`) where the old lexical compare did not — harmless (column only holds `0..100`/NULL
and the flag is display-only).

### Environment note
This box runs **node v24**, which natively strips types and loads `.ts` as ESM, bypassing
`ts-node/register` (`require is not defined in ES module scope`) — it breaks *all* unit
tests here, not this change. Workaround used locally: `NODE_OPTIONS=--no-experimental-strip-types`.
CI uses the project's pinned node; run `npm run test:unit` there for the green confirmation.

## Post-deploy data cleanup (ELEVATED — separate, named approval)
The fixed code only recomputes a hub's flag on its next event, so the ~2,340 stale flags
won't clear promptly. **After the fix is live**, propose this one-time scoped correction.
Verify the count with the SELECT first; the UPDATE requires the user to name the resource
("go — tbl_alarms lowBattery cleanup") and is then recorded in `docs/applied-changes.md`.

```sql
-- 1) count first (expect ~2,340)
SELECT COUNT(*) FROM tbl_alarms
WHERE controller = 1 AND lowBattery = 1 AND CAST(batteryStatus AS UNSIGNED) > 15;

-- 2) the correction
UPDATE tbl_alarms SET lowBattery = 0
WHERE controller = 1 AND lowBattery = 1 AND CAST(batteryStatus AS UNSIGNED) > 15;
```
Open question for the cleanup: whether to also clear the `readStatus=0` unread markers set
alongside each spurious flip (the code sets `readStatus=false` when it raises `lowBattery`).

## Verification checklist
- [x] SQL-layer: `CAST(batteryStatus AS UNSIGNED) <= 15` excludes `'100'`, includes `'13'`.
- [x] Unit: new test passes; full suite 79 passing (node-24 workaround).
- [x] Regression-guard: SAFE, display-only blast radius confirmed.
- [ ] CI: `npm run test:unit` green on canonical node.
- [ ] Post-deploy: trigger `updateAlertOnHub` on a healthy 100% hub (e.g. a
      `scripts/diag/mqtt-diag.py --mode battery` poll, propose→go) → confirm `lowBattery`
      stays/returns to 0; re-run the blast-radius `COUNT(*)` and confirm it falls.
- [ ] Data cleanup applied (named approval) + logged in `docs/applied-changes.md`.

See also: MEMORY `hub-lowbattery-string-compare-systemic-bug`.
