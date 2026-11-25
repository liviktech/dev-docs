# 2026-10-27 - raw-glass-reports (backend)

## Summary

The client asked for two raw glass reports (Raw Glass Batch Consumption and Cutting Utilisation, and a Reusable Offcut Register), and for the operator to record wastage and offcut before a piece moves on from Cutting. Most of the data was already saved by the stock cutting code, so this PR adds read-only report endpoints on top of it. It also adds a small "cutting entry" (finished glass, wastage, offcut length and width) that is saved on the work order piece's Cutting stage, and gives every stock offcut a readable Offcut ID.

The code compiles (`tsc --noEmit` passes). The two report queries were run against the dev database (RUBINE GLASS company) and returned rows, but with the new migration columns stood in as empty values, because migration 096 was **not run** by me. The cutting entry save and the migrations were **not checked live** here.

**Posting the entered offcut into stock is not included on purpose** - see "Decisions made" below. The entered offcut is only recorded and shown in the report. This was a choice, not a miss.

## What changed

| File | Change |
|---|---|
| `src/database/migrations/094_inventory_offcut_code.sql` (new) | Adds `offcut_code` (OFC-000001 and so on) to `public.inventory_lot_offcut`, numbers the existing rows in creation order, adds a sequence so new offcuts get a code from the column default, and adds a unique index. Adds only, changes nothing that exists. |
| `src/database/migrations/096_work_order_process_cutting_entry.sql` (new) | Adds `finished_sqm`, `wastage_sqm`, `offcut_length` and `offcut_width` to `production.work_order_process`, with a check that none is negative. |
| `src/modules/report/inventory/inventory-report.schema.ts` (new) | Zod rules for the two report queries (dates, work order, batch, specification, location, status, paging). |
| `src/modules/report/inventory/inventory-report.repository.ts` (new) | All SQL for Batch Consumption, Offcut Register and one lean filter options call (ids and names only). |
| `src/modules/report/inventory/inventory-report.service.ts` (new) | Turns rows into the report shape and works out waste and cutting utilisation. |
| `src/modules/report/inventory/inventory-report.controller.ts` (new) | Handlers for `batch-consumption`, `offcut-register` and `filter-options`. |
| `src/modules/report/report.routes.ts` | Mounts `/reports/inventory` with the existing `reports` VIEW right. The Sales Income and FIR routes from upstream are kept next to it. |
| `src/modules/production-scan/production-scan.schema.ts` | Adds the cutting entry (finished glass and offcut length and width required, wastage optional) to the complete stage body. |
| `src/modules/production-scan/production-scan.service.ts` | Refuses to complete the Cutting stage without the entry and passes it on. Other stages are unchanged. Returns the saved entry with each stage. |
| `src/modules/production-scan/production-scan.repository.ts` | Saves the entry columns when a stage completes and reads them back with the stage. |
| `src/modules/production-scan/production-scan.controller.ts`, `production-scan.types.ts` | Pass the entry through and add the `cuttingEntry` type on a stage. |

## Why

The client's Finished Goods and Raw Glass spec lists two reports. The easy part was the reports, because the cutting code already saves one stock movement per piece with the piece size, the leftover size and the source lot. The real work was deciding what the Production Date, the Manufacturing Stage and the offcut statuses mean, and finding the places where the existing data does not line up.

## How

**Batch Consumption**: One row per production date, work order, line, stage and raw glass batch. Raw glass consumed is the area of the sheet (or the offcut) a piece was cut from. Finished area is the piece size and offcut area is the recorded leftover. Waste is the entered wastage, or the consumed area minus finished minus offcut (never below 0) when none was entered. Cutting utilisation is finished area divided by consumed area. A reversed cut (a return row points at it) is left out. Cuts made before piece sizes were saved show only the consumed area and leave the other numbers empty.

**Production Date and stage come from the work order execution log**: The date is the day the piece's Cutting stage Started (Asia/Kolkata), and the stage is its work center name. A stock cut that is not linked to a piece takes the work order's earliest Cutting start. If the work order never reached Cutting, the approval date is used.

**Offcut Register**: One row per stock offcut row. Generated Date uses the same Cutting start rule, through the piece that made the offcut. Consuming Work Order is every work order that has a non reversed stock movement against that offcut.

**Offcut status is worked out, not stored**: Available when a free piece remains. Reserved when none is free but an approved work order has taken a piece whose Cutting stage is not complete. Consumed otherwise.

**Paging**: Both reports take `page` and `limit` and return one page plus the full count. With no `page` they return every row, which the downloads use.

## Decisions made

**Offcut status is calculated in the report, not changed in the stock code.** The other choice was to move the stored status to Reserved at work order approval and Consumed after Cutting. I did not pick it because the stock code counts pieces per offcut row and reverses them when a work order changes, and a second status there could break those counts. The stored status still becomes CONSUMED at approval, as before.

**The entered offcut is recorded only.** It is not added to stock. Stock offcuts still come from the cutting calculation at work order approval.

**Cutting is the only stage that asks for the entry.** Raw glass is only consumed there, and offcut and finished glass do not apply to the other stages.

**Wastage is optional.** When it is empty the report works the waste out from the cut.

**An unlinked older stock cut uses the earliest Cutting start of its work order.** Latest start is the other choice and it is still open (see "Next steps").

**Migration 094 for the Offcut ID is a new file, not an edit of 079.** 079 was already applied, so editing it would change nothing in the database. A column default from a sequence means the offcut insert code did not need to change.

## Bugs found while testing (already there before, not fixed here)

**1. Older work orders have one stock record for the whole order.** Before per piece cutting, the stock deduction was written once with no piece and no sizes. For these, the report has no piece to read a date or sizes from. It uses the work order's earliest Cutting start and shows only the consumed area. It affects work orders approved before per piece cutting, such as WO-20261015-152.

**2. The stored offcut status becomes CONSUMED at work order approval, not after cutting.** This is the existing stock behaviour. Any screen that reads the stored status sees Consumed early. Only the new Offcut Register shows the Reserved then Consumed flow.

**3. A chain of cuts saves every leftover as an offcut row.** When each piece is cut from the leftover of the previous cut, every step becomes its own offcut row and all but the last are marked CONSUMED straight away. The Offcut Register therefore lists many used rows next to the one leftover that is still free.

## Notes

- Run `npm run migrate` and restart the backend. Migration 096 is needed before the production piece page and Complete Stage work, because the stage query now reads the new columns. The Batch Consumption report also reads them.
- Upstream added `094_fir.sql`, so two migrations now start with 094. If the runner tracks by full file name this is fine. Rename `094_inventory_offcut_code.sql` to a later number only if your runner needs unique numbers (every step in it is safe to run again).
- The stash merge with the new Sales Income and FIR routes was resolved by keeping both sets of routes.
- Production Date uses the Started time in the Asia/Kolkata time zone, the same rule the production reports use.

## Next steps

- Decide whether older whole order stock cuts should use the earliest or the latest Cutting start of the work order (earliest now, latest would match the piece 2 screen in WO-20261015-152).
- Decide if the entered offcut should be posted into stock as a real offcut.
- Add a Scrapped offcut status and a scrap action on the Stock screen (none exists today).
- Define a minimum reusable offcut size. It goes in the two marked places in the cutting code.
- Consider hiding Consumed rows by default in the Offcut Register.

## Test plan

**Automated tests (no new tests added, type check passing):**
- [x] `npx tsc --noEmit` in the backend: passes.
- [ ] No unit or API tests were written for the new report queries or the cutting entry rule. The backend has no test file for the report module to extend.

**Manual checks (local):**
- [x] Ran the Batch Consumption query through the service against the dev database for RUBINE GLASS (new columns stood in as empty): it returned rows with the stage and date taken from the execution log, and WO-20261015-152 gave 20/10/2026.
- [x] Ran the Offcut Register query the same way: it returned 17 offcuts with Reserved and Available statuses and a Generated Date from the log.
- [ ] Migrations 094 and 096 were not run by me. Migration 094 looked already applied in the dev database when checked.
- [ ] Completing a Cutting stage with and without the entry through the API was not checked against a database.
