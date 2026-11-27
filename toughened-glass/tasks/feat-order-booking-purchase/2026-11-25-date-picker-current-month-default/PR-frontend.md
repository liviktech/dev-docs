# 2026-11-25 - date-picker-current-month-default (frontend)

## Summary

Before this PR, every screen with the shared date range picker in its header opened on "All dates", while the dashboard opened on the current month. The picker's Reset also went back to "All dates". This made the list screens and reports load every record by default, and the KPI cards (which follow the date range only) showed totals for all time.

This PR makes every desktop screen, inventory tab and report that has the picker in its header open on the **current month**: the 1st of the month up to today (India time), the same as the dashboard's "This Month". Reset in the picker now goes back to the current month. It is a frontend-only change.

All changes were tested in the browser by the author. Automated results are listed in the Test plan.

**The Department Daily report is not changed on purpose** - see "Decisions made" below. This was a choice, not a miss.

## What changed

| File | Change |
|---|---|
| `src/lib/date-range.ts` | New `currentMonthRange()`: the 1st of this month to today in India time. It reads the date each time it is called, so a screen opened after the month turns starts on the new month. Returns the same object for the same day. |
| `src/features/enquiries/enquiries-page.tsx` | URL filter default `fromDate` / `toDate` set to the current month. The header picker gets `defaultRange`, so Reset returns to it. |
| `src/features/pi-status/pi-status-page.tsx` | Same. The existing `getDefaultDateRange` now returns the current month. |
| `src/features/final-invoices/final-invoices-page.tsx` | Same (URL filter default and picker Reset). |
| `src/features/dispatch/dispatch-page.tsx` | Same. |
| `src/features/fir/fir-page.tsx` | Same. |
| `src/features/sales-visits/sales-visits-page.tsx` | Same. |
| `src/features/expense-management/expenses-page.tsx` | Same. Its "has active filters" check and Reset now compare against the current month instead of empty dates. |
| `src/features/archive/archive-page.tsx` | Starting filters use the current month. Picker Reset goes back to it. |
| `src/routes/orders.tsx` | Order Booking: the header range starts on the current month unless the URL already carries `from` / `to`. Picker Reset goes back to it. |
| `src/routes/work-orders.tsx` | Work Orders list: the range read from the URL falls back to the current month. Both header pickers get `defaultRange`. |
| `src/features/production-plan/production-plan-panel.tsx` | Production Plan tab (under Work Orders): the range starts on the current month. |
| `src/features/inventory/stock-tab.tsx` | URL filter default set to the current month. Picker Reset goes back to it. |
| `src/features/inventory/purchase-tab.tsx` | Same. |
| `src/features/inventory/customer-material-tab.tsx` | From and To start on the current month. |
| `src/features/inventory/movements-tab.tsx` | The range starts on the current month. |
| `src/features/procurement/procurement-list-state.ts` | Indent and Purchase Order lists: the starting filters use the current month. |
| `src/features/procurement/procurement-ui.tsx` | The header picker gets `defaultRange`. |
| `src/features/reports/*` (15 report files) | Each report's starting filters use the current month and its `dateRange` option on `ReportLayout` passes `defaultRange`: Enquiry Follow-Up, FIR, Batch Consumption, Offcut Register, Order Status, Sales Rep Achievement, Department, Production Detail, Creditors, Daily Invoice, Debtors, Location Sales, PI Tracking, Sales Rep History and Visit Register. |
| `src/features/reports/department-daily/department-daily-report.tsx` | no real change |

No backend, database or API change.

## Why

The dashboard already opens on the current month and users expected the same from every screen with the same picker. Opening on all dates also made lists and KPI cards heavier and less useful by default.

The easy part was the helper and the per-screen default. The real work was finding how each screen holds its date state: URL-backed lists, plain local state, a route-level state (Order Booking) and the shared report page hook. The Reset buttons and "active filters" checks had to be checked on each so that the new default did not look like a changed filter.

## How

**One helper**: every screen calls `currentMonthRange()` for its starting dates and passes the same value to the picker as `defaultRange`. The picker disables Reset while the range equals the default and restores it when clicked.

**URL-backed lists** (Enquiries, PI Status, Final Invoices, Dispatch, FIR, Sales Visits, Expenses, Stock, Purchase): the default is passed to the existing URL filter hook. A value equal to the default is left out of the URL, so a clean `/enquiries` link opens on the current month. A different range is written to the URL and survives a refresh.

**Order Booking and Work Orders**: the range is read from the URL when present and falls back to the current month otherwise. Order Booking's list writes the range into the URL, so its URL now carries `from` and `to` even for the default month.

**Reports**: the starting filters get the month dates. The filter-bar Reset on the reports already kept the date range, so it works as before.

**KPI cards**: they follow the date range only (earlier change), so they now show current-month figures by default.

**Kept the same on purpose**: the picker component and its quick ranges, the filter-bar Reset buttons (they leave the dates alone), the mobile PWA lists, and which reports have a picker.

## Decisions made

**Department Daily report still opens on today.** It is a daily summary and already started on today, so it is not "all dates". Not picked: moving it to the month.

**There is no "All dates" any more.** Reset returns to the current month, so there is no way back to an unlimited range. To see everything, pick "This financial year" or a custom range. Not picked: adding an "All dates" quick option, because it was not asked for.

**The default is read when a screen opens, not when the app loads.** A tab left open across a month change starts the new month on the next screen opened. Not picked: a module-level constant, which would keep the month the app loaded in.

**Reports without a range picker are untouched**: Sales Rep Performance (Day / Week / Month / Year period), Price List (one effective date) and Work Order Progress.

**Mobile PWA lists and the Tally check view are not changed.**

## Bugs found while testing (already there before, not fixed here)

None.

## Notes

- Existing bookmarks and shared links that carry their own `from` / `to` keep working; only links without dates change what they show.
- Order Booking's URL now always carries the dates, so the address bar is longer than before. This does not change behaviour.
- The default month runs from the 1st to today, not to the end of the month, the same as the dashboard.

## Next steps

- Decide whether to add an "All dates" quick range to the picker.
- Decide whether the Department Daily report should also open on the month.
- Decide whether the mobile PWA lists should get the same default.

## Test plan

**Automated tests:**
- [x] Frontend `npx tsc -b`: no errors.
- [ ] Frontend unit tests: not run for this change. The frontend has no test runner of its own (`tsx` is not installed there).

**Manual checks (local):**
- [x] Each list screen, inventory tab and report listed above opens on the current month, and the data it loads follows that range.
- [x] Reset in the picker returns to the current month and is disabled while the range is already the default.
- [x] Picking another range works, and on URL-backed screens it survives a refresh.
- [x] KPI cards follow the default range.
- [x] Department Daily opens on today; Sales Rep Performance, Price List and Work Order Progress show no change.
- [x] Additionally, a headless-browser pass by the assistant on a throwaway local backend confirmed the picker label and the first dated request on 17 screens and 16 report views were all 01 Nov ,  25 Nov 2026, and Today then Reset on Enquiries, Order Booking, Customer Material, Debtors and Expenses returned to the month.
