# 2026-10-30 - sales-rep-targets-and-reports (backend)

## Summary

Sales reps had no targets, so there was no way to measure how a rep was doing against a goal. This PR adds three targets to the Sales Rep master (target amount per month, target enquiry per day, target enquiry per month), a built-in Direct sales rep for enquiries and orders that have no rep, the data for a new Sales Rep Performance report (Enquiry Wise and Amount Wise), and three more columns on the Proforma Invoice Tracking report (Sales Executive, Enquiry Date, Advance Amount Paid). It also changes the Orders figure on the old performance report (now shown as Sales Rep Achievement) to count every order booked from the rep's enquiries, not only the ones that came from a visit.

Checked with the TypeScript build only (no errors in the touched files). The migrations were not run and the new endpoints were not called against real data, see "Notes" below.

**Running the migrations is not included on purpose** - see "Notes" below. This was a choice, not a miss.

## What changed

| File | Change |
|---|---|
| `src/database/migrations/103_sales_rep_targets.sql` (new) | Adds `target_amount_per_month` (NUMERIC 14,2), `target_enquiry_per_day` and `target_enquiry_per_month` (INTEGER) to `master.sales_rep`. All three are nullable in the table so old rows stay valid. |
| `src/database/migrations/104_sales_rep_direct_row.sql` (new) | Adds `is_system` (BOOLEAN, default false) to `master.sales_rep` and inserts a `DIRECT` row for every company that does not have one yet. |
| `src/modules/master/sales-rep/sales-rep.schema.ts` | The three targets are required on create (not negative, the two enquiry targets must be whole numbers). On update they are optional but, when sent, follow the same rules. |
| `src/modules/master/sales-rep/sales-rep.types.ts` | Row, DTO and table item types gain the three targets and `isSystem`. |
| `src/modules/master/sales-rep/sales-rep.repository.ts` | Reads and writes the three targets. Adds `DIRECT_SALES_REP_CODE` and `ensureDirectSalesRep` (creates the Direct row if a company does not have it). The paged list calls `ensureDirectSalesRep` first. The "active reps" list used by dropdowns no longer includes system rows. |
| `src/modules/master/sales-rep/sales-rep.service.ts` | Maps the new fields. Adds `assertNotSystem`: the Direct row cannot be deleted, deactivated or recoded. Its name and targets can still be edited. |
| `src/modules/enquiry/enquiry.repository.ts` | An enquiry saved with no sales rep is now saved as `DIRECT` on create and on update. |
| `src/modules/report/sales-rep-targets/` (new, 4 files) | Controller, repository, schema and service for the new Sales Rep Performance report. Three endpoints, see "How". |
| `src/modules/report/report.routes.ts` | Mounts `/sales-executives`, `/enquiry-wise` and `/amount-wise` under `/reports/sales-rep-performance`. The existing route on that path is untouched. |
| `src/modules/report/sales-visit/sales-visit-report.repository.ts` | The orders count per rep is now one figure, `orders`: every non-cancelled order booked from one of the rep's enquiries in the period. The visit-only count and the total count are gone. |
| `src/modules/report/sales-visit/sales-visit-report.service.ts` | Row type and mapping use the single `orders` field. Removed `totalOrders`, `ordersFromVisits` and `ordersWithoutVisits`. |
| `src/modules/report/sales-visit/sales-visit-report.schema.ts` | Comment only (the report is now called Sales Rep Achievement). No real change. |
| `src/modules/report/sales-income/sales-income-report.repository.ts` | The PI tracking query now returns `sales_executive`, `enquiry_date` and `advance_amount`. It joins the PI to its order, the order to its enquiry, the enquiry to its rep, and reads the PI's Advance payment term. |
| `src/modules/report/sales-income/sales-income-report.service.ts` | Returns `salesExecutive`, `enquiryDate` and `advanceAmount` on each PI tracking row. |

## Why

The business wanted to compare each rep against a monthly sales target and a daily enquiry target, and to see the same report for enquiries and orders that come in without a rep. Putting those under a built-in Direct rep means Direct can carry targets like any other rep, and the reports do not need special cases for "no rep". The PI tracking columns were asked for so the team can see who owns a PI, when its enquiry came in, and how much advance was agreed.

The easy part was the three columns and the master fields. The real work was the day-by-day backlog sheet and deciding how an order finds its rep.

## How

**Targets on the master**: stored on `master.sales_rep`. The API accepts them as numbers. The screen works out the per day and per month enquiry targets from each other, so the backend only stores what it is given.

**Direct sales rep**: code `DIRECT`, `is_system = true`. Migration 104 creates it for existing companies, and `ensureDirectSalesRep` creates it for a company added later. `assertNotSystem` stops delete, deactivate and recode, but not edits to the name or targets. Dropdowns that pick a rep never offer it.

**Enquiry Wise** (`GET /reports/sales-rep-performance/enquiry-wise?month=YYYY-MM`): one sheet per rep, one row per working day (Sundays left out). Target for a day = the daily target plus the backlog carried from the day before. Backlog = target minus achieved. The month starts fresh. Days that have not come yet show no numbers. An inactive rep only shows if they have enquiries in that month. The carry forward is worked out on the server so the screen, PDF and Excel all agree.

**Amount Wise** (`GET /reports/sales-rep-performance/amount-wise?salesRepId=&year=`): one row per month from January to the current month (the whole year for a past year, nothing for a future year). Basic Value is the glass amount before charges and tax, and SQM is the chargeable area. Cancelled orders are left out. An order belongs to the rep of the enquiry it was booked from. An order with no enquiry, or an enquiry with no rep, counts under Direct.

**Sales executives list** (`GET /reports/sales-rep-performance/sales-executives`): the active reps, with Direct last, for the Amount Wise dropdown.

**PI tracking columns**: Sales Executive is the enquiry's rep name, or Direct when there is no enquiry or no rep. Enquiry Date is empty when the PI has no enquiry. Advance Amount Paid is read from the PI's payment term named Advance: a fixed value is capped at the PI total, a percentage is worked out on the PI total (rounded to 2 decimals), and a PI with no Advance term shows nothing.

**Kept the same on purpose**: PI tracking order (newest PI date first), its paging, and the existing visit register and Sales Rep Achievement filters.

## Decisions made

**Direct is a real row in the sales rep table, not a report-only label.** The other option was to count "no rep" in the reports without a row. A real row lets Direct carry its own targets and show in the master list, and it is created by the program so nobody has to set it up.

**Advance Amount Paid is the agreed advance, not money actually received.** I did not find a place where advances received against a PI are recorded. So this shows the Advance term of the PI. If there is a payments table I missed, this should switch to it.

**The old performance report's order figures were merged into one.** The report used to show total orders, orders from visits and orders without visits. It now shows one `orders` figure that counts every order from the rep's enquiries, including email and phone enquiries. This is a change in what the number means, so the Orders figure can now be higher than Visits Converted to Orders.

**Old enquiries that have no rep were not changed.** Only new saves default to Direct. The reports already treat a missing rep as Direct, so they still count correctly.

## Notes

- The migrations (103 and 104) were not run by me, because the database is the shared dev database. Run `npm run migrate` and restart the backend before using the new reports. Until then the sales rep list and the new reports will fail.
- Nothing was run against real data. The October numbers have not been compared with the sheet the business uses.
- If the Orders column on the Sales Rep Achievement report shows "undefined" in the browser, the backend is still running the old code (it still sends the old field names). Restart it.
- The Sales Rep Achievement report only counts orders whose enquiry has a rep. Orders with no rep appear only on the new Performance report, under Direct.

## Next steps

- Backfill old enquiries that have no sales rep to `DIRECT`, if the business wants the master data to match the reports.
- Switch Advance Amount Paid to the amounts actually received if there is a place where they are recorded.
- Compare the Enquiry Wise and Amount Wise numbers for October against the business sheet.

## Test plan

**Automated tests (no test files added or changed):**
- [x] TypeScript build (`npx tsc --noEmit`): no errors.

**Manual checks (local):**
- [ ] Run migrations 103 and 104, then open the sales rep list - not done, the migrations are left for the user to run.
- [ ] Create a rep without a target, expect a validation error - not done, nothing was called against the running API.
- [ ] Try to delete or deactivate the Direct row, expect an error message - not done.
- [ ] Call the three Sales Rep Performance endpoints and the PI tracking endpoint with real data - not done. The code was checked by reading and by the type check only.
