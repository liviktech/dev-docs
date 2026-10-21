# 2026-10-21 - downtime-stats-and-report (backend)

## Summary
Downtime could be recorded on a stage, but nobody could see how much time was lost. This PR adds downtime totals to the Production Line list, the piece list of a work order and the per product, per stage progress. It also adds a new `GET /production-scan/downtimes` endpoint that returns every downtime with filters and a summary per work center. Open downtimes (no end time) are counted up to now.

No database changes. The new queries read the existing `production.work_order_process_downtime` table. `npx tsc --noEmit` passes. The new endpoint and queries were not called by hand, see "Test plan".

## What changed
| File | Change |
|---|---|
| `src/modules/production-scan/production-scan.types.ts` | Added `downtimeMinutes` and `openDowntimeCount` to `ScanListItem` and `ScanStageProgress`, and `downtimeMinutes` and `openDowntime` to `ScanPieceRow`. Added the stats row types, `DowntimeReportFilter`, `DowntimeReportRow` and the report response types (`ScanDowntimeReportEntry`, `ScanDowntimeWorkCenterSummary`, `ScanDowntimeReport`). |
| `src/modules/production-scan/production-scan.schema.ts` | Added `downtimeReportQuerySchema` (optional `from` and `to` as YYYY-MM-DD, `workCenterCode`, `workOrderId` as a uuid, `status` as `open` or `closed`). |
| `src/modules/production-scan/production-scan.repository.ts` | Added `listDowntimeStatsForWorkOrders`, `listDowntimeStatsForItems`, `listDowntimeStatsByProductStage` and `listDowntimeReport`. All share one `DOWNTIME_MINUTES` expression, where an open downtime runs to `CURRENT_TIMESTAMP`. The report query is limited to 2000 rows, newest first. |
| `src/modules/production-scan/production-scan.service.ts` | `listScanWorkOrders`, `getWorkOrderPieces` and `getWorkOrderStageProgress` now load the stats and add the downtime fields. Added `getDowntimeReport` (filters, per entry duration, summary per work center sorted by most minutes first). Added small helpers `minutesOf`, `startOfDay` and `startOfNextDay`. |
| `src/modules/production-scan/production-scan.controller.ts` | Added the `getDowntimeReport` handler. |
| `src/modules/production-scan/production-scan.routes.ts` | Added `GET /downtimes` with query validation. It allows the same viewers as the progress route (`PROGRESS_VIEWERS`) plus `master_work_center` VIEW. |

## Why
The shop floor already records breakdowns, but managers could not tell which work order, piece or work center was losing time. The Work Orders and PI Status tables, the piece list, the Work Center page and a report all need the same numbers, so the totals come from the backend in one place instead of each screen adding up entries itself.

## How
**Downtime totals**: Each stats query sums the minutes of the downtimes and counts the ones with no end time. They are grouped per work order, per piece, or per product and work center. The service reads the sum as a string and turns it into whole minutes with `minutesOf` (a missing value becomes 0).

**Progress endpoint**: Each stage of each product now also carries `downtimeMinutes` and `openDowntimeCount`. The stage is matched on the order booking item id and the work center code.

**Report endpoint**: `from` is the first day shown and `to` is the last day, included whole. The service turns `to` into the start of the next day and the query uses `start_time < to`, so the last day is not cut off. The days are read as UTC. Each entry has the work order, party, piece barcode, product, stage and who recorded it (the last updater, or the creator if never updated).

**Kept the same on purpose**: The existing downtime create and update routes, and the rule that blocks completing a stage with an open downtime, are not touched.

## Decisions made
**Downtime on rejected pieces is counted.** The progress stats count downtime for every piece of the product, rejected ones included, because the time was lost either way. The piece counts next to them still skip rejected pieces as before.

**Totals are worked out in SQL, not from the downtime list.** The list pages do not load every downtime, so summing on the database keeps them light.

**The report is capped at 2000 rows.** There is no pagination yet. The summary is built from the returned rows, so it only covers those 2000.

## Notes
- `from` and `to` day boundaries use UTC (`T00:00:00.000Z`). A downtime near local midnight can land on the neighbouring day if the users are not in UTC.
- Work orders without any downtime return `downtimeMinutes: 0` and `openDowntimeCount: 0`, so the new fields are always present.

## Next steps
- Add pagination or a stricter date limit to the report if the 2000 row cap is hit in real use.

## Test plan
**Automated tests (no new tests):**
- [x] `npx tsc --noEmit`: no TypeScript errors.
- [ ] No unit or integration tests were added for the stats queries or the report. The module has no tests and none were written here.

**Manual checks (local):**
- [ ] `GET /production-scan/downtimes` and the new stats on the list, piece and progress responses were not called directly with a REST client. They are meant to be seen through the frontend screens in the frontend PR.
- [ ] The UTC day boundary and the 2000 row cap were not tested with real data.
