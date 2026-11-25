# 2026-11-03 - sales-rep-history-job-type (backend)

## Summary

A party only kept its current sales person, so when Company A moved from John to Peter there was no record of who had it before or for how long. PI Status, Final Invoice and Dispatch records also had no job type, so the screens could not tell a Without Material order (the customer brings the glass) from a normal one. This PR adds a sales person history table and a report endpoint for it, returns `job_type` on the PI, Final Invoice and Dispatch queries, and fixes the Stock list, which was showing the customer's own glass lots and crashed when you typed in its search box. It also lets the finish stage accept any active storage location. The same working tree has dispatch and cutting changes for customer glass that were not part of this task, they are listed in the table as they appear in the diff.

`npx tsc --noEmit` passes on the current tree. Migration 115 and the new SQL were **not run** against a database from here, and no API call was made. No automated tests were run or added.

**Several raw sheet sizes per Without Material product ("Add Pieces") is not included on purpose** - see "Decisions made" below. This was a choice, not a miss.

## What changed

| File | Change |
|---|---|
| `src/database/migrations/115_party_sales_rep_history.sql` (new) | Creates `master.party_sales_rep_history` (party, sales person, `assigned_from`, `assigned_to`) with a check that the end is not before the start, an index per party, and a unique index so a party has at most one open (current) row. Backfills one row per party that already has a sales person, starting from the party's creation date. |
| `src/modules/master/party/party.repository.ts` | New `recordSalesPersonChange`. On create it starts the first history row. On update it locks the party row, compares the old and new sales person, and when they differ closes the open row and opens a new one. Saving a party without changing the sales person adds nothing. |
| `src/modules/report/sales-rep-history/sales-rep-history-report.schema.ts` (new) | Query filters: from date, to date, party, sales person, status (ALL, CURRENT, PAST) and the usual paging. |
| `src/modules/report/sales-rep-history/sales-rep-history-report.repository.ts` (new) | One row per assignment with party, sales person, from, to and days of service (counting both end days). The date filter keeps assignments that overlap the range. Also lists the sales people that appear in the history, for the filter dropdown. |
| `src/modules/report/sales-rep-history/sales-rep-history-report.service.ts` (new) | Maps the rows to the API shape (`partyCode`, `partyName`, `salesPerson`, `assignedFrom`, `assignedTo`, `days`, `isCurrent`). |
| `src/modules/report/sales-rep-history/sales-rep-history-report.controller.ts` (new) | Two handlers, the report and its filter options. |
| `src/modules/report/report.routes.ts` | Mounts `GET /reports/sales-rep-history` and `GET /reports/sales-rep-history/filter-options`, both behind the existing reports VIEW permission. |
| `src/shared/utils/order-job-type-sql.util.ts` (new) | `orderJobTypeSql(companyIdExpr, orderCodeExpr)`, one SQL subquery that gives an order's job type: `WITHOUT MATERIAL` when every product is, `MIXED` when only some are, else `WITH MATERIAL`, and null when the order has no products. Used by the three modules below so the rule lives in one place. |
| `src/modules/proforma-invoice/proforma-invoice.repository.ts` | The three PI queries now return `job_type`. |
| `src/modules/proforma-invoice/proforma-invoice.types.ts` | Optional `job_type` on the list row type. |
| `src/modules/final-invoice/final-invoice.repository.ts` | `job_type` added to both the full list select and the lean table select (the by id fetch reuses the full select). |
| `src/modules/final-invoice/final-invoice.types.ts` | Optional `job_type`. |
| `src/modules/dispatch/dispatch.repository.ts` | `job_type` added to the list select and the table select. The gate pass select now lists the customer's balance glass only once it was marked returned, and reads the saved cutoffs as `jsonb` so `COALESCE` no longer mixes `json` and `jsonb`. |
| `src/modules/dispatch/dispatch.types.ts` | Optional `job_type` on both row types. |
| `src/modules/inventory/inventory.repository.ts` | `listLots` (the Stock list) now excludes customer lots with `NOT l.is_customer_material`. Removed the `sp.name ILIKE` condition from its search, because no table is called `sp` in that query. |
| `src/modules/production-scan/production-scan.repository.ts` | `activeLocationExists` now accepts any active storage location. It used to require the type FINISHED_GOODS. |
| `src/modules/production-scan/production-scan.service.ts` | Error text for a bad finish location now says "Choose an active storage location for the finished glass". |
| `src/modules/dispatch/dispatch.controller.ts` | New `getCustomerBalance` handler. |
| `src/modules/dispatch/dispatch.routes.ts` | New `GET /dispatches/:id/customer-balance` (dispatch VIEW permission). |
| `src/modules/dispatch/dispatch.schema.ts` | `returnBalanceGlass` (boolean, default false) on the approve dispatch body. |
| `src/modules/dispatch/dispatch.service.ts` | `approveDispatch` marks the invoice's customer lots as returned when `returnBalanceGlass` is true. New `getCustomerBalance`. |
| `src/modules/inventory/customer-lot-dispatch.repository.ts` (new) | `listCustomerBalanceForInvoice` (customer lots on an invoice that still hold uncut sheets or cutoffs) and `returnCustomerLotsForInvoiceInTx` (marks each one returned through the existing `returnCustomerLotInTx`). |
| `src/modules/work-order/work-order-inventory.service.ts` | The cutting run now records each cut's leftover (`recordLeftovers: true`) so a later piece can be cut from it. The cutting plan gets two new fields, `offcutReused` (the leftover is already planned for another piece, so there is no offcut to enter) and `customerSupplied` (the glass is the customer's own). |
| `src/modules/work-order/work-order-stock.service.ts` | The stock simulation also uses `recordLeftovers: true`. |
| `src/modules/production-scan/production-scan.types.ts` | Four comment lines only, no real change. |
| `src/modules/work-order/work-order.types.ts` | Four comment lines only, no real change. |

## Why

The business asked to keep a history of which sales executive had a customer and for what period. The current party row cannot answer that because it is overwritten, and the other tables (enquiries, visits) only say who handled one record, not who owned the account. So a new table was needed. The job type on PI, Final Invoice and Dispatch records was needed so the screens can show a Without Material badge. The Stock fixes bring the list back in line with what migration 112 promised, that customer glass never appears in Rubine's stock. The easy part was the history table. The real work was making sure the history stays in step with the party row in the same transaction.

## How

**Sales person history**: Each assignment is one row. When the sales person on a party changes, the old open row gets `assigned_to = NOW()` and a new row starts at the same `NOW()`, so the periods have no gap. A cleared sales person closes the open row and starts no new one. The history is stored by name because the party's sales person is still free text.

**Report**: The query joins the history to the party and orders by party name, then newest assignment first. Status CURRENT means `assigned_to IS NULL`, PAST means it is set. Without `page` it returns every row, which the Excel and PDF downloads use, the same pattern as the FIR report.

**Job type**: `orderJobTypeSql` looks at the products of the order (`sales.order_booking_item`) for each PI, Final Invoice or Dispatch and classifies the order. The match ignores case, spaces, underscores and dashes, the same as `isWithoutMaterial` elsewhere. The value is passed through to the API as `job_type`, nothing else about those records changed.

**Stock list**: The customer lot filter is one added condition, so the list, the card view, the table view and the in stock and depleted counts all drop customer lots together.

**Finish stage location**: The check now only asks that the location exists, is active and belongs to the company. The frontend dropdown shows every active location to match.

## Decisions made

**History is stored by name, not linked to the sales rep master.** The party's `sales_person` is free text today, so a typo or a rename shows up as a different person. Linking it to `master.sales_rep` is a bigger change and was not asked for.

**Existing parties start from their creation date.** Earlier reassignments were never recorded and cannot be recovered, so the backfill creates one row per party from `created_at`.

**One shared job type SQL helper.** The same rule was needed in three modules. A shared helper avoids three copies drifting apart. The values are `WITHOUT MATERIAL`, `MIXED`, `WITH MATERIAL` or null.

**The Stock list bug was fixed here, not just flagged.** The missing filter put customer lots in Rubine's stock, which the design says must not happen. The `sp.name` search error was a leftover reference to a table that is not in the query, so removing it is safe, product names are still searched through `p.name`.

**Finish stage accepts any active location.** It was Finished Goods only. The team asked for all locations to show, so the backend check had to match, otherwise saving would fail.

**Several raw sheet sizes per product ("Add Pieces") is not included.** The idea was to let a Without Material product carry extra pieces of other sizes. A product stores one raw sheet size and gets exactly one customer lot, and work order cutting, dispatch and the balance return all assume that. Doing it properly means several lots per product. The first steps were started and then dropped, nothing from it is in this PR (there is no migration 116).

## Notes

- Several type files have a comment line `Without Material (customer glass) order` that sits above fields it does not describe (four in `production-scan.types.ts`, four in `work-order.types.ts`). They look like leftovers and have no effect. Left as they are.
- The dispatch customer balance endpoint, `returnBalanceGlass`, the gate pass change, the leftover recording and the new cutting plan fields were already in the working tree. They are listed from the diff as they stand and were not changed by this task.
- Migration 115 is append only and safe to run again, it uses `IF NOT EXISTS` and `ON CONFLICT DO NOTHING`.

## Next steps

- Run `npm run migrate` to apply 115, then open PI Status, Final Invoice and Dispatch to confirm the lists still load with the new `job_type` column.
- Decide whether the party sales person should become a link to the sales rep master.
- Decide how Add Pieces should work for stock (several lots per product) before building it.
- Remove the stray `Without Material (customer glass) order` comment lines.

## Test plan

**Automated tests (none run, type check passes):**
- [x] `npx tsc --noEmit`: passes with no errors on the current tree.
- [ ] `tests/sales-rep-summary.test.ts` and the rest of the backend tests: not run. No test was added for the history report, the party history writes or `job_type`.

**Manual checks (local):**
- [ ] Apply migration 115, assign John to a party, then change it to Peter, and check `master.party_sales_rep_history` has two rows with John closed at the same moment Peter opened - not run, needs the migration on a database.
- [ ] Save a party without changing the sales person and check no new row appears - not run, same reason.
- [ ] Call `GET /reports/sales-rep-history` with each filter and check paging and the download (no `page`) - not run, same reason.
- [ ] Call the PI, Final Invoice and Dispatch lists and check `job_type` for a with material order, a without material order and a mixed order - not run, same reason.
- [ ] Open Stock with a Without Material order saved and check its lot is not listed, then type a search term and check there is no error - not run, needs a running backend and data.
- [ ] Complete the last stage of a piece with a Raw Glass location and check it saves - not run.
