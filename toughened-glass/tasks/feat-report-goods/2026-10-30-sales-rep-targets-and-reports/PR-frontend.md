# 2026-10-30 - sales-rep-targets-and-reports (frontend)

## Summary

The Sales Rep screen had no targets and the reports could not show how a rep was doing against a goal. This PR adds the three targets to the Sales Rep form and list (all required), shows the built-in Direct rep without Delete or Deactivate, renames the old performance report to Sales Rep Achievement and trims its columns, adds a new Sales Rep Performance report (Enquiry Wise and Amount Wise), and adds Sales Executive, Enquiry Date and Advance Amount Paid to the Proforma Invoice Tracking table in the order the team asked for.

Checked with the TypeScript build only (it passes). Nothing was opened in the browser, see "Notes" below.

## What changed

| File | Change |
|---|---|
| `src/features/sales-reps/sales-rep-dialog.tsx` | Adds Target Amount per Month, Target Enquiry per Day and Target Enquiry per Month. All three are required (a number, not negative, and a whole number for the two enquiry targets) and carry the red star. The per day and per month enquiry targets fill each other in from the working days of the month (Sundays left out). |
| `src/features/sales-reps/api.ts` | Maps the three targets and `isSystem` from the list response. |
| `src/features/sales-reps/types.ts` | `SalesRep` gains the three targets and `isSystem`. |
| `src/routes/admin.sales-reps.tsx` | Three new columns in the table and three new rows in the details panel. The Direct row has no Delete item and its status switch is disabled. Search placeholder shortened to "Search". |
| `src/features/reports/performance/sales-rep-achievement-report.tsx` (new) | The old Sales Rep Performance report, moved here and renamed Sales Rep Achievement. Removed the Total Orders and Orders Without Visits columns, and "Orders from Visits" is now just "Orders" (every order from the rep's enquiries). Excel and PDF follow the same columns, including the PDF totals row. |
| `src/features/reports/performance/api.ts` (new) | Types and fetch functions for the new report: enquiry wise, amount wise and the sales executive list. |
| `src/features/reports/performance/sales-rep-performance-report.tsx` | Rewritten as the new Sales Rep Performance report. Enquiry Wise has a rep dropdown, a month dropdown and a daily sheet (Date, Target Per Day, Achieved, Backlog, Performance %). Amount Wise has a sales executive dropdown, a year dropdown and one row per month (Month, Target, Basic Value, SQM). Both have Excel and PDF downloads. |
| `src/features/reports/sales-visit/api.ts` | Row type uses the single `orders` field in place of `totalOrders`, `ordersFromVisits` and `ordersWithoutVisits`. |
| `src/features/reports/components/report-table.tsx` | New optional `showAllRows` prop, passed to the shared table, so a month of days or a year of months shows on one page. Default is off, so every other report is unchanged. |
| `src/features/reports/registry.ts` | Registers the new Sales Rep Achievement screen next to Sales Rep Performance. |
| `src/features/reports/report-menu.ts` | Adds the "Sales Rep Achievement" menu item under Sales Rep. |
| `src/features/reports/sales-income/api.ts` | PI tracking row type gains `salesExecutive`, `enquiryDate` and `advanceAmount`. |
| `src/features/reports/sales-income/pi-tracking-report.tsx` | Three new columns. Final order: PI Number, PI Date, Enquiry Date, Party, Sales Executive, PI Amount, Advance Amount Paid, Status. Excel and PDF use the same order, and the PDF right-aligns the two amount columns. An empty enquiry date or advance shows a dash. |
| `src/features/inventory/components/purchase-dialog.tsx` | Build fix, not part of the reports work. The thickness list callback was named `MM` but its text used `mm`, which stopped `npm run build` with TS2552. Renamed the callback value to `mm`. |
| `src/features/inventory/components/receive-stock-dialog.tsx` | Same one-word build fix. |
| `src/features/pwa/inventory/mobile-lot-form.tsx` | Same one-word build fix. |
| `src/features/attendance/attendance-page.tsx` | Search placeholder shortened to "Search". No other change. |
| `src/features/attendance/daily-attendance-tab.tsx` | Search placeholder shortened to "Search". No other change. |

## Why

The business wanted rep targets on the master and two new views to measure reps against them, a daily enquiry sheet and a monthly sales view. The PI tracking columns were asked for so the team can see the owner, the enquiry date and the agreed advance on each PI without opening it. The rest follows from that, for example the old report needed a new name so the two reports are not confused.

The easy part was the columns and the form fields. The real work was the Enquiry Wise sheet and keeping the table, Excel and PDF in the same shape.

## How

**Targets form**: the same two-way fill as the rest of the form. Typing the per day target fills the per month target, and the other way round, using the working days of the chosen month. Values are sent as numbers on save.

**Direct rep**: the list sends `isSystem`. The page uses it to hide Delete and disable the status switch. The backend refuses those actions as well, so this only avoids showing a button that cannot work.

**New Performance report**: the numbers come from the server, including the backlog carried from one day to the next. The screen only shows them, so the table, PDF and Excel match.

**Column alignment**: text columns are left aligned, amounts are right aligned (as in the other PI tracking columns), and the first column is left aligned. Enquiry Date is left aligned like PI Date so the two dates line up.

**Kept the same on purpose**: the other reports, paging and filters of the PI tracking report, and every table that does not pass `showAllRows`.

## Decisions made

**Enquiry Date is left aligned.** The table alignment guide says dates are centered. In this report PI Date was already left aligned, so I matched it. This is a one-line change if the team wants dates centered.

**The Achievement report lost its other order columns, not just one.** First "Orders Without Visits" was removed, then "Total Orders". "Orders" now counts all orders from the rep's enquiries (visit, email, phone), so it replaces both. The visit-only figure no longer shows anywhere.

**Search placeholders became "Search" on the Sales Rep list and the two attendance tabs.** This is a small text change only.

## Notes

- Nothing was opened in the browser. The screens were checked by reading the code and by the TypeScript build only.
- The backend migrations (103 and 104) must be run and the backend restarted before these screens work. If Orders on Sales Rep Achievement shows "undefined", the backend is still running the old code.
- Advance Amount Paid shows the agreed Advance term of the PI, not money actually received.

## Next steps

- Open the new reports in the browser with real data and compare October against the business sheet.
- Decide if dates in the PI tracking table should be centered, as the alignment guide says.

## Test plan

**Automated tests (no test files added or changed):**
- [x] TypeScript build (`npx tsc --noEmit -p .`): no errors.
- [x] `npm run build` equivalent (`npx tsc -b`): no errors. It had 3 errors ("Cannot find name 'mm'") in the three inventory files above before the fix.

**Manual checks (local):**
- [ ] Open Master > Sales Reps, add a rep with an empty target, expect a message on each empty field - not done, the screen was not opened.
- [ ] Open the Direct row, expect no Delete item and a disabled status switch - not done.
- [ ] Open Reports > Sales Rep > Sales Rep Achievement and check the columns, Excel and PDF - not done.
- [ ] Open Sales Rep Performance, pick a rep and month, then a sales executive and year - not done.
- [ ] Open Proforma Invoice Tracking and check the column order, Excel and PDF - not done.
