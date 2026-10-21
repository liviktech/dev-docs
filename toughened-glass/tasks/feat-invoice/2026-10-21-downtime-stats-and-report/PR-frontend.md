# 2026-10-21 - downtime-stats-and-report (frontend)

## Summary
Downtime could be recorded but its effect was not visible anywhere. This PR shows the lost time on the Work Orders and PI Status Progress cell, the progress dialog (per stage), the piece list of a work order, the stage stepper, the execution log, the mobile piece screens and the Work Centers page. A downtime that is still open shows in red as Open, a closed one shows in amber with the total time.

`npx tsc -b --noEmit` passes. The screens were not clicked through by me for this PR, see "Test plan".

**A Downtime report page and a delivery risk flag are not included on purpose** - see "Next steps" below. The backend report endpoint is ready, but no page uses it yet except the Work Centers page. This was a choice, not a miss.

## What changed
| File | Change |
|---|---|
| `src/features/production-scan/downtime-utils.ts` (new) | Helpers: `formatMinutes` (45m, 2h 15m), `downtimeMinutesOf`, `totalDowntimeMinutes`, `hasOpenDowntime` and `isoDay`. |
| `src/features/production-scan/components/downtime-list.tsx` (new) | `OpenDowntimeBadge` (red Open pill) and `DowntimeList`, a box listing each downtime of a stage with reason, start, end, who recorded it and how long. |
| `src/features/production-scan/types.ts` | Added the downtime fields on the list item, stage progress and piece row, and `downtimes` on the log entry. Added the report types and `DowntimeReportParams`. |
| `src/features/production-scan/api.ts` | Added `fetchDowntimeReport`, which skips empty filters. |
| `src/features/production-scan/queries.ts` | Added `downtimeReportQueryOptions` (refetches every 30 seconds so open downtimes keep counting). |
| `src/features/production-scan/stage-utils.ts` | `logEntriesOf` now passes each stage's `downtimes` into the log entry. |
| `src/features/production-scan/components/work-order-progress.tsx` | The Progress cell shows "Downtime open" in red or the total lost time in amber. The progress dialog has a new Downtime column per stage. |
| `src/features/production-scan/components/execution-log-table.tsx` | Added Downtime (right aligned like the other time columns; the entries are listed in the stage panel, not in a hover tooltip) and Net Time (completed time minus downtime) columns. |
| `src/features/production-scan/components/completed-stage-panel.tsx` | Shows the `DowntimeList` under the execution log of a completed stage. |
| `src/features/production-scan/components/stage-stepper.tsx` | A small timer badge on a stage that has downtime (red when open, amber when closed). A completed stage that can be clicked now also has a thin green ring so it looks clickable. The first stage now has a short line before it and the last stage a short line after it, so the stepper has visible start and end lines. |
| `src/features/production-scan/work-order-pieces-table.tsx` | Added a Downtime column to the pieces table, with a total on each product row. |
| `src/features/pwa/work-orders/mobile-piece-detail.tsx` | Shows downtime under each stage step, and Downtime, Net time and the entry list in the stage log. |
| `src/features/pwa/work-orders/mobile-work-order-pieces.tsx` | A piece card shows a downtime pill (red when open). |
| `src/routes/work-orders.tsx` | Passes `downtimeMinutes` and `openDowntimeCount` to `WorkOrderProgress`. |
| `src/features/pi-status/components/pi-status-table.tsx` | Same two props passed to `WorkOrderProgress`. |
| `src/routes/admin.work-centers.tsx` | Added a "Downtime (30 Days)" column and a "Downtime (Last 30 Days)" row in the details dialog (total, entry count, open count and the 5 newest entries). Only loaded for users with VIEW on `work_order`, `barcode_scanner` or `master_work_center`. |
| `src/routes/work-orders_.$workOrderId.tsx` | Pulled the open piece navigation into an `openPiece` function. The loading, error, empty and table states are unchanged, only re-indented inside a wrapper. No real change in behaviour. |

## Why
Once downtime was being recorded, the next question was how much time each work order, piece and work center was losing, and whether a breakdown is still open. The numbers are shown where people already look (Work Orders, PI Status, piece list, stage page, mobile, Work Centers) instead of on a separate page.

## How
**Open versus closed**: A downtime with no end time counts up to the current time, so it keeps growing. It is shown in red with an Open badge. Closed downtime is shown in amber.

**Net Time**: Completed time minus the total downtime of that stage, never below zero. It only shows for a completed stage.

**Where the numbers come from**: The Work Orders, PI Status, piece list and progress dialog use the totals the backend now sends. The stage stepper, execution log and mobile stage views add up the `downtimes` list of each stage on the screen with the helpers in `downtime-utils.ts`.

**Work Centers page**: One call to the report endpoint with `from` set to 30 days ago. The result is grouped by work center code for the column, and the first 5 entries of the work center fill the dialog.

## Notes
- `isoDay` builds the date in UTC, because the backend reads report days as UTC. This keeps the 30 day window on the Work Centers page in step with the backend. Between midnight and the UTC offset in the morning (for example 00:00 to 05:30 in India), "today" is still the previous UTC date, so the window can be one day behind at that hour. That is fine for a rolling 30 day total.

## Next steps
- Build a Downtime report page on top of the report query, and add a "delivery at risk" flag on the Work Orders list. 

## Test plan
**Automated tests (no new tests):**
- [x] `npx tsc -b --noEmit`: no TypeScript errors.
- [ ] No unit tests were added for `downtime-utils.ts` (for example `formatMinutes` and `deliveryRiskOf`). None were written here.

**Manual checks (local):**
- [ ] The Work Orders, PI Status, pieces list, stage page, mobile piece screens and Work Centers page were not opened in a browser for this PR, so the layout and the colours were not looked at.
- [ ] The Work Centers column for a user without VIEW access was not checked. The query is disabled for them in code (`enabled: canViewDowntime`).
