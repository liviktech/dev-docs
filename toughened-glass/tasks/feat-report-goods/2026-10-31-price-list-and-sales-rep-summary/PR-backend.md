# 2026-10-31 - price-list-and-sales-rep-summary (backend)

## Summary

Two things were missing. Product rates could only hold one price per glass type and thickness, and saving a product deleted and re-created every rate, so old prices were lost and there was no way to plan a price change for a future date. Also, the Sales Rep reports only showed one rep at a time, so there was no single table with every rep over a few months.

This PR turns product rates into a dated price history, adds a new Price List report that reads it, and adds two "all reps" summary endpoints to Sales Rep Performance. Existing endpoints keep the same shape.

The new SQL was **not run against a database** and the new endpoints were **not called live**, see "Test plan".

## What changed

| File | Change |
|---|---|
| `src/database/migrations/110_product_rate_history.sql` (new) | Adds `price_component` (BASE, ENERGY_SURCHARGE, POLISH, ROUGH_GRINDING) and `updated_at` to `master.product_rate`, with a check constraint. Drops the old one price per product, thickness and type unique index. Adds a unique index of one active price per date, and a lookup index for the "latest price on or before a date" query |
| `src/modules/master/product/product.schema.ts` | New `priceComponentEnum`. `rateInputSchema` takes `priceComponent` (defaults to BASE) |
| `src/modules/master/product/product.types.ts` | New `PriceComponent` type, added to the rate row and rate item types |
| `src/modules/master/product/product.repository.ts` | Rates are returned with `priceComponent` and only active ones are returned. Create and update no longer write `effective_to`. Update no longer deletes rates: rates the form did not send are switched off, rates sent with an id are updated, and the rest are inserted |
| `src/modules/report/price-list/price-list-report.schema.ts` (new) | Query validation, one optional `asOf` date (YYYY-MM-DD) |
| `src/modules/report/price-list/price-list-report.repository.ts` (new) | Two queries: all active dated BASE glass prices, and the fixed amount charges from the products' Charges tab |
| `src/modules/report/price-list/price-list-report.service.ts` (new) | Picks the price in force on the date for each product, glass type and thickness, builds the history list (current, scheduled, previous), and finds the energy, polish and rough grinding charges |
| `src/modules/report/price-list/price-list-report.controller.ts` (new) | Thin handler for the endpoint |
| `src/modules/report/report.routes.ts` | New `GET /reports/price-list` and two new sales rep summary routes. All need the Reports VIEW right |
| `src/modules/report/sales-rep-targets/sales-rep-targets-report.schema.ts` | New `summaryQuerySchema`: `from` and `to` as YYYY-MM, From not after To, at most 12 months. Exports `monthIndex` and `MAX_SUMMARY_MONTHS` |
| `src/modules/report/sales-rep-targets/sales-rep-targets-report.repository.ts` | New `findMonthlyAmountsAllReps`: basic value and SQM per rep per month in one query |
| `src/modules/report/sales-rep-targets/sales-rep-targets-report.service.ts` | New `summaryMonths`, `getEnquiryWiseSummary` and `getAmountWiseSummary` |
| `src/modules/report/sales-rep-targets/sales-rep-targets-report.controller.ts` | Two new handlers for the summary routes |
| `tests/sales-rep-summary.test.ts` (new) | Tests for the summary query rules and the month list |

## Why

The business keeps a price list sheet and a sales rep summary sheet by hand. Prices change on set dates, and the team needs to enter a new price in advance and still see what was charged before. The summary sheets show every rep side by side over several months, which the one rep at a time endpoints could not give.

The easy part was the routes and the schema. The real work was the rate sync in the product update, because it used to delete everything and now has to keep history.

## How

**Price in force**: for a date, the price is the active row with the greatest `effective_from` on or before that date. `effective_to` is no longer used or written. A row dated after the date is "SCHEDULED", the one in force is "CURRENT", the rest are "PAST".

**Rate sync on product update**: rows whose id is not in the sent list are set `is_active = FALSE` (never deleted). A row sent with an id is updated in place. If that update finds no row, or no id was sent, it is inserted. The create path just inserts.

**Price List report**: one table per glass type, one row per thickness, one column per product. A glass type with no price in force on the date is hidden. Only BASE prices make the columns. Energy surcharge, polish and rough grinding come from the Charges tab. The charge master has no fixed codes for them, so they are found by looking for ENERGY, POLISH or GRIND in the charge code or name.

**Today for the report** is taken in India time (Asia/Kolkata), not the server clock, so a price starts at the start of the day there.

**Enquiry wise summary**: per rep per month, target is the daily target times working days (Sundays out, and for the running month only up to today). Achieved is enquiries created. Backlog is target minus achieved, and it does not carry over to the next month, same as the day by day sheet.

**Amount wise summary**: per rep per month, basic value and SQM from non cancelled orders by booking date. An order takes the rep of its enquiry. No enquiry or no rep means Direct. The monthly target is shown once per rep.

**Months in the future** are dropped, so asking for a range that ends later than this month stops at this month, and a range fully in the future returns empty lists.

**Kept the same on purpose**: the existing `enquiry-wise` and `amount-wise` endpoints and their responses are untouched. The reps list is the same `findRepTargets` they use.

## Decisions made

**History stays in `master.product_rate`.** No separate history table was added. Rows are never deleted and the date makes each one unique, so a new table would only have repeated the same data.

**Inactive reps are hidden unless they have data.** In both summaries an inactive rep (not the Direct one) only appears if they have enquiries or orders in the range.

**Charge amounts are one value for every row.** The Price List shows a single energy, polish and rough grinding amount, taken from the first product (by name) that sets it. It does not vary per thickness. The frontend and migration also support these as dated BASE style components, but the report does not read them yet, see "Next steps".

## Notes

- Migration 110 must be run before the product screens or the Price List work. Without it, `price_component` and `updated_at` do not exist and the product rate queries fail.
- `npm test` only picks up `src/**/*.test.ts`, so the new file in `tests/` is not part of that script. It was run directly (see Test plan).
- Existing rate rows get `price_component = 'BASE'` from the column default, so old data keeps working.

## Next steps

- Run migration 110 on a copy of real data. If a product already has two active rows for the same thickness and type on the same date, the new unique index will fail to build.
- Decide whether energy surcharge, polish and rough grinding should read the dated `price_component` rows instead of the Charges tab.
- Move `tests/sales-rep-summary.test.ts` under `src/` or widen the `npm test` glob so it runs with the rest.

## Test plan

**Automated tests (7 passing):**
- [x] `tests/sales-rep-summary.test.ts` "sales rep summary query": accepts a month range, rejects From after To, allows at most 12 months, rejects a bad month. 4 cases pass.
- [x] `tests/sales-rep-summary.test.ts` "summaryMonths": lists months and marks the running one, stops at the current month, crosses a year end. 3 cases pass. Run with `node --import tsx --test tests/sales-rep-summary.test.ts`.
- [x] TypeScript build (`npx tsc --noEmit`): no errors.
- [ ] No tests were written for the Price List service or the product rate sync.

**Manual checks (local):**
- [ ] Run migration 110 and save a product with a current and a scheduled price - not done, the database was not used.
- [ ] Call `GET /reports/price-list` with and without `asOf` - not done.
- [ ] Call the two summary endpoints for a few months and compare with the business sheet - not done.
