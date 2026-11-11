# 2026-10-27 - raw-glass-reports (frontend)

## Summary

The Reports page had no raw glass reports, and the operator could complete a Cutting stage without recording what the cut left behind. This PR adds a "Raw Glass" group to the Reports menu with two reports (Batch Consumption and Offcut Register), and a "Cutting entry" box that opens when Complete Stage is pressed on the Cutting stage. It also makes the report filter boxes a little wider and writes area headings as "(sqm)" on the existing production reports.

The code compiles (`tsc -b` passes). The requester opened the Batch Consumption screen and the Cutting entry box locally (screenshots), and the report showed the work order, stage and a production date. Saving the entry from the box and the Offcut Register screen were **not checked live** by me, and the report downloads were not tried.

**Posting the entered offcut into stock is not included on purpose** - see the backend PR. The box only records the values and the report shows them. This was a choice, not a miss.

## What changed

| File | Change |
|---|---|
| `src/features/reports/report-menu.ts` | Adds the "Raw Glass" group with Batch Consumption and Offcut Register. The sidebar menu builds itself from this list. |
| `src/features/reports/registry.ts` | Attaches the two new screens. The Sales Income and FIR screens from upstream are kept. |
| `src/features/reports/inventory/api.ts` (new) | Calls and types for the new report endpoints and the filter options call. |
| `src/features/reports/inventory/inventory-options.ts` (new) | One call fills the work order, batch, specification and location dropdowns. Holds the Available, Reserved and Consumed status list. |
| `src/features/reports/inventory/batch-consumption-report.tsx` (new) | Batch Consumption screen: date range, work order, batch and specification filters, server paging, Reset button, and Excel and PDF downloads that include every matching row. |
| `src/features/reports/inventory/offcut-register-report.tsx` (new) | Offcut Register screen: date range, specification, location, status and source work order filters, server paging, Reset, Excel and PDF. |
| `src/features/production-scan/components/cutting-entry-dialog.tsx` (new) | The Cutting entry box: finished glass, offcut length and width (required, 0 means none) and wastage (optional). |
| `src/features/production-scan/scan-detail-page.tsx` | Complete Stage on the Cutting stage opens the box first, and sends the entry with the complete call. |
| `src/features/production-scan/components/completed-stage-panel.tsx` | Shows the saved cutting entry above the execution log of a completed Cutting stage. |
| `src/features/production-scan/api.ts`, `queries.ts`, `types.ts` | Send the entry with the complete call and add the entry type. |
| `src/features/reports/components/report-select-filter.tsx`, `report-month-filter.tsx`, `report-period-filter.tsx` | Filter boxes are 11rem wide instead of 9rem so longer labels are not cut off. |
| `src/features/reports/production/department-report.tsx`, `production-detail-report.tsx` | Area headings now read "(sqm)" instead of "(sq m)". No other change. |

## Why

The client's Finished Goods and Raw Glass spec asks for these two reports and for wastage and offcut to be entered before a piece moves to the next stage. The reports reuse the existing report layout, table, paging and Reset button, so most of the work was the two new screens and the entry box.

## How

**Report screens**: They follow the same pattern as the other server paged reports. The table shows one page, and the Excel and PDF buttons ask the server for every matching row. Glass specification, work order line and stage are shown in Pascal case with the shared `PascalText` component.

**Cutting entry box**: The Complete Stage button calls the same handler as before, and for a Cutting stage it opens the box instead of completing. The box passes the values to the same handler. The stale stage check still runs before the complete call.

**Offcut status column**: It shows the status the backend works out (Available, Reserved or Consumed). Nothing is calculated in the browser.

## Decisions made

**The entry box only appears for the Cutting stage.** Other stages complete as before.

**Wastage is optional and the other three fields are required.** When wastage is left empty the report works the waste out from the cut.

**The box has no date field.** The work order execution log already shows Started and Completed times, so the report reads those instead of a second date.

**The Batch Consumption screen has no Source, Finished Pieces or Cutting Completed Date column.** These were removed at the client's request.

## Notes

- Run the backend migrations first (`npm run migrate`, migration 096 in the backend), or the production piece page and Complete Stage will fail.
- The Offcut Register lists many Consumed rows for a chain of cuts on one big sheet. That is how the stock data is saved today.
- Merge conflicts with the new Sales Income and FIR report entries in `report-menu.ts` and `registry.ts` were resolved by keeping all groups.

## Next steps

- Check the Offcut Register and the report downloads in the browser with real data.
- Confirm the Cutting entry saves and shows on the completed stage after the migration is run.
- Decide whether older whole order records should use the earliest or latest Cutting start for the Production Date (see the backend PR).

## Test plan

**Automated tests (no new tests added, type check passing):**
- [x] `npx tsc -b` in the frontend: passes.
- [ ] No component or unit tests were written for the new screens or the box. There is no test setup for report screens in this repo to extend.

**Manual checks (local):**
- [x] Reports > Raw Glass > Batch Consumption opened and filtered by work order WO-20261015-152: one row with production date 20/10/2026, stage Cutting, batch and specification (seen by the requester in a screenshot).
- [x] The Cutting entry box opened from Complete Stage on a piece with the four fields, a required star on three of them and the Save button disabled until they are filled (seen by the requester in a screenshot).
- [ ] Saving the entry and seeing it on the completed stage was not checked, because it needs backend migration 096.
- [ ] Offcut Register screen, Excel and PDF downloads were not opened.
