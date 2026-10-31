# 2026-10-31 - price-list-and-sales-rep-summary (frontend)

## Summary

The product Rates tab only allowed one flat price per thickness and type, so a price change meant overwriting the old one. There was no Price List report, and Sales Rep Performance only showed one rep at a time. Separately, the New Purchase button on the work order stock section could open with a sheet size that was too small for the pieces.

This PR changes the Rates tab to a current price plus an optional scheduled new price, shows price status in the product view, adds a Price List report, adds "All reps summary" layouts to Sales Rep Performance (with Excel and PDF), and fixes the sheet size passed to New Purchase.

The screens were **not clicked through in the browser**, see "Test plan". The backend (migration 110 and the new endpoints) must be in place first.

## What changed

| File | Change |
|---|---|
| `src/features/products/types.ts` | New `PriceComponent` type. Rate type has `price_component`. Save payload rates take `thicknessId`, `glassTypeId` and `priceComponent`, and `effectiveTo` is gone |
| `src/features/products/api.ts` | Maps `priceComponent` from the API, defaulting to BASE |
| `src/features/products/product-dialog.tsx` | Rates tab is now one line per component, thickness and type with Current Price, a new price and its future Effective Date. `groupRateLines` and `flattenRateLines` convert between the API's one row per dated price and these lines. New validation for the dates and prices. Modal widened to `max-w-6xl` and the table scrolls sideways if needed |
| `src/features/products/product-view-modal.tsx` | Rates table shows Price For, Thickness, Type, Rate and a Status (Current, Scheduled or Previous) instead of Effective To |
| `src/features/reports/price-list/api.ts` (new) | Types and fetch for `GET /reports/price-list` |
| `src/features/reports/price-list/price-list-report.tsx` (new) | The Price List report: an as-of date, a table per glass type, charge columns, a history view, Excel and PDF export |
| `src/features/reports/report-menu.ts` | New Price List group in the Reports menu |
| `src/features/reports/registry.ts` | Registers the Price List screen |
| `src/features/reports/performance/api.ts` | Types and fetches for the two summary endpoints |
| `src/features/reports/performance/sales-rep-summary-reports.tsx` (new) | The Enquiry Wise and Amount Wise "All reps summary" tables, with a From and To month, Excel and PDF export |
| `src/features/reports/performance/sales-rep-performance-report.tsx` | Enquiry Wise and Amount Wise are now tabs. A dropdown in each picks the one rep layout or the all reps summary. Total row figures are bold and darker |
| `src/features/reports/components/report-table.tsx` | Supports grouped column headers (a month over its sub columns), and the pagination footer is hidden when all rows are shown |
| `src/components/ui/data-table.tsx` | Header cells pass `colSpan`. A column with `meta.divider` gets a left border |
| `src/features/reports/export-xlsx.ts` | New optional `groupHeaders` that adds a merged heading row above the headers |
| `src/lib/pdf/report-table-pdf.ts` | New optional `groupHeaders` that adds a heading row above the headers |
| `src/features/order-booking/work-order-stock-section.tsx` | New Purchase now passes a sheet size that fits every piece, see "How" |

## Why

Prices change on set dates and the business needs to enter the next price early, still see what applied before, and print a price list for any date. The summary tables replace a sheet the team keeps by hand. The New Purchase fix is for the case where the held lot is smaller than a piece, so the purchase form opened with a size that could never cut the order.

## How

**Rates tab lines**: the tab shows the price in force today and one optional new price with a future date. Older and further out prices are kept out of the form but sent back on save, so history is not lost. If the current price is edited, the old price stays dated as it was and the new one starts today. A new price needs both its date and its amount, and the date must be after today.

**Product view**: the status is worked out per line. The latest price dated today or earlier is Current, later ones are Scheduled, the rest are Previous.

**Price List**: one table per glass type with a row per thickness and a column per product. Each cell shows the price in force on the chosen date, and the history can be shown with each dated price marked Current, Scheduled or Previous. Energy, polish and rough grinding amounts sit in their own columns.

**Summary tables**: the month range is limited to 12 months, same as the server. Each month is a group heading over its sub columns (Target, Achieved, Backlog for enquiries, Basic Value and SQM for amounts). The running month reads "(till <day>)". Group headings are carried into the Excel and PDF files.

**New Purchase sheet size**: a held lot's size is used only if every piece fits it (a piece can be turned). Otherwise the size is the smallest sheet that holds all the pieces, with each piece turned so its long side runs along the width. If there are no pieces, no size is passed.

**Kept the same on purpose**: the one rep Enquiry Wise and Amount Wise screens still use their old endpoints and columns. The `pagination` behaviour of report tables is unchanged when `showAllRows` is off.

## Decisions made

**One price per line in the form, not a full history grid.** Showing every dated price would make the tab long and easy to break. The form edits the current price and one scheduled price, and keeps the rest hidden but safe.

**Edited current price becomes a new entry from today.** The old price is not overwritten so reports for past dates still show what was charged.

**Mode dropdown inside tabs.** The old view dropdown became tabs, and a smaller dropdown inside each tab picks the layout, so the one rep screens did not have to change.

## Notes

- Several files only had line ending warnings from Git (`product-dialog.tsx`, `product-view-modal.tsx`, `sales-rep-performance-report.tsx`). The summary above is based on the diff with line endings ignored.
- Migration 110 on the backend must be run first. Without it the Rates tab and Price List will fail.

## Next steps

- Open the Rates tab, the Price List and the summary tables in the browser with real data and compare against the business sheets.
- Add a way to schedule a price for the energy, polish and rough grinding components in the Rates tab if the business wants dated values for them.

## Test plan

**Automated tests (no test files added or changed):**
- [x] TypeScript build (`npx tsc --noEmit -p tsconfig.app.json`): no errors.

**Manual checks (local):**
- [ ] Edit a product, change the current price, add a new price with a future date and save - not done, the screen was not opened.
- [ ] Open the product view and check the Current, Scheduled and Previous status - not done.
- [ ] Open Reports > Price List, change the as-of date, and export Excel and PDF - not done.
- [ ] Open Sales Rep Performance, switch to All reps summary on both tabs, pick a month range, and export Excel and PDF - not done.
- [ ] Open a work order where a piece is larger than the held sheet and click New Purchase, expect the sheet size to fit the piece - not done.
