# 2026-10-16 - final-invoice-sheet-and-pi-charges (backend)

## Summary
This PR covers all the uncommitted backend changes in the repo. First, opening any Final Invoice preview failed: `GET /final-invoices/:id` returned a 500, and the e-invoice and e-way bill calls for the same invoice failed with it. The code reads `order_booking_item_id` from `invoice.proforma_invoice_charge`, but no migration ever created that column, so Postgres answered `column "order_booking_item_id" does not exist`. A migration adds it. Second, the Final Invoice charge and discount maths now handles older PIs whose order level charges were only stored in the PI header total, so the invoices of a PI add up to the PI. Third, the PI list and detail now return the item id on each charge row.

The migration was applied to the shared Neon dev database with `npm run migrate`. After that, `findFinalInvoiceById` was called directly for the failing invoice and returned its 1 item, 3 charges and 2 taxes. The other code changes were checked by a type check only (clean). I did not retest any screen in a browser.

## What changed
| File | Change |
|---|---|
| `src/database/migrations/066_proforma_invoice_charge_item_link.sql` (new) | Adds a nullable `order_booking_item_id UUID` column to `invoice.proforma_invoice_charge`, plus the index `idx_pi_charge_item`. Safe to run more than once (`IF NOT EXISTS`). |
| `src/modules/final-invoice/final-invoice.repository.ts` | New `getUnitemisedPiCharges`: the part of the PI header `charge_amount` that no charge row backs. New `getPiDiscountBase`: the PI `basic_amount` (falls back to items plus charges), used as the base the order discount is worked out on. `getFinalInvoiceChargesBreakdown` takes a new `includeUnitemised` flag (default true) and adds those charges to the order level charges, with one extra "Order Charges" line, prorated like the others. `findFinalInvoiceById` now builds the charges and discount through a small `computeCharges` helper and tries a few combinations until the result matches the invoice's stored taxable amount. |
| `src/modules/final-invoice/final-invoice.service.ts` | `createFinalInvoice` and `getEligibleItemsForInvoice` take the discount from `getPiDiscountBase`. `piOrderCharges` now includes the unitemised charges and no longer falls back to the PI total charges. |
| `src/modules/proforma-invoice/proforma-invoice.repository.ts` | `listProformaInvoices` and `findProformaInvoiceById` return `orderBookingItemId` on each charge row. In `createProformaInvoice` the order level charge insert had one more value than placeholders (a `KEY:` charge key). That extra value is removed. |

## Why
The fabrication charges work (item level charges copied onto the PI) started using the new column, but the schema change was never added, so every Final Invoice detail read failed. The fix belongs in the schema, not in the queries, because the column is the intended design.

PIs saved before order level charge rows were stored only have the total in the PI header. For those PIs the Final Invoices added up to less than the PI, and the discount was taken on a different base than the PI and order booking use. The frontend PI drawer also needed the item id on each charge row to tell product charges from order level charges.

## How
**Nullable column**: existing rows get NULL, which the code already treats as an order level charge. Nothing is rewritten.

**Unitemised charges**: the gap is the PI header `charge_amount` minus the sum of its charge rows. Gaps under half a paisa count as zero. The gap is shared out by item value like any other order level charge.

**Only for invoices made with them**: `findFinalInvoiceById` first computes the old way (no unitemised charges, old discount base). If that does not match the invoice's stored `taxable_amount` within 0.05, it tries the other combinations and keeps the first that matches. So invoices created before this change keep showing the same figures they had.

**Discount base**: the PI `basic_amount`, which is the figure the order booking and PI take the discount from. PI level charges added on top are not discounted.

**Kept the same on purpose**: the GST maths, the proration formula and the stored invoice totals are unchanged.

## Decisions made
**Fixed the missing column with a migration, not by changing the queries.** Removing the column from the queries would have brought the preview back but dropped the per item fabrication charges the feature is meant to show.

**Tried the combinations against the stored taxable amount instead of changing old invoices.** The other option was a data fix on old rows. Matching on read leaves the saved data alone and works for both old and new invoices, but the read now does up to four calculations when the first one does not match.

## Bugs found while testing (already there before, not fixed here)
**1. Item charge rows could not be saved while the column was missing.** The PI create code skips failures when copying item charges, so PIs created before this fix may have no item level charge rows. Their Final Invoices fall back to the order booking charges.

**2. The frontend `fetchFinalInvoiceDetail` hides errors.** It logs the error and returns null, which is why the screen showed an empty sheet. The frontend PR now shows an error panel. The helper itself was not changed.

## Notes
- The migration changed the shared dev database. Every other environment (staging, production) needs `npm run migrate` before deploy, or the same 500 will happen there.
- The `/e-invoice` and `/e-way-bill` 500s in the console were not checked one by one. They are expected to clear with the detail endpoint, but that is not confirmed.
- The extra value removed in `createProformaInvoice` was inside a try block that swallows errors, so it is likely that order level charge rows were not being saved before. That was not tested against a database.

## Next steps
- Run `npm run migrate` on every other environment before deploying.
- Decide whether older PIs should have their item level charges backfilled.
- Test the unitemised charges path with a PI that only has a header charge total.

## Test plan
**Automated tests (none added):**
- [x] Type check (`tsc --noEmit`): no errors.
- [ ] No automated tests exist for the Final Invoice charge and discount maths.

**Manual checks (local):**
- [x] Ran `npm run migrate` on the dev database: only 066 was pending and it applied.
- [x] Called `findFinalInvoiceById` directly for invoice `2416a81e-d699-41ff-bc22-97508b23a597`: it returned 1 item, 3 charges and 2 taxes. Before the migration it failed with `column "order_booking_item_id" does not exist`.
- [ ] Open the Final Invoice preview in the app and check the e-invoice and e-way bill calls no longer return 500. Not checked by me.
- [ ] Create a Final Invoice from a PI with a header only charge total, and check the invoices add up to the PI. Not checked.
