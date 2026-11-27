# 2026-11-12 - per-item-gst (backend)

## Summary
Before this PR, every Proforma Invoice (PI) and Final Invoice was taxed at one rate for the whole document. The rate was a number typed on the item, or a fallback of 18%, and the e-Invoice and e-Way Bill used a default HSN of `70071900` when an item had none. Now each item links to a Tax Profile (the GST rate) and has its own HSN code. Each PI line and Final Invoice line is taxed at its own item's rate, and the invoice tax is the sum of the line taxes. The backend does all the maths, and the screens only show what it returns. If an item has no HSN or no Tax Profile, the request stops with a message naming the item. It never guesses a rate. Older PIs and Final Invoices are not changed and keep the old single-rate way. Only the type check was run (passes). The migration was applied on a local database but has not been run on live, and nothing was checked end to end on live data.

## What changed
| File | Change |
|---|---|
| `src/database/migrations/134_per_item_gst_tax_profile.sql` (new) | Adds `master.item.tax_profile_code` (FK to Tax Profile) and fills it from the typed rate where exactly one active profile has that percentage. Adds line columns (item code, HSN, rate, line value, taxable value, CGST/SGST/IGST, tax) to the PI and Final Invoice item tables. Adds header columns (tax method, interstate flag, CGST/SGST/IGST, round off, PI discount) to both invoice tables. Moves raw materials off the Sales Account and adds a check so they cannot go back. Adds a check that a finished good on Sales has an HSN and a Tax Profile, created as `NOT VALID` so old rows are not checked. |
| `src/shared/gst/line-tax.ts` (new) | `computeInvoiceTax`. Shares order charges and discount over the lines by value, taxes each line at its own rate, rounds per line, splits CGST/SGST or IGST, and rounds the grand total to a whole rupee with the difference as round off. |
| `src/shared/gst/line-tax-resolver.ts` (new) | Finds the item behind each order booking item (product, thickness, colour first, then the product's own item). Only active finished goods on the Sales Account are candidates. Collects every problem into one 400 (`PI_ITEM_TAX_MISSING`). |
| `src/modules/proforma-invoice/pi-tax.service.ts` (new) | Works out the PI tax for a set of order bookings and decides same state or other state from the buyer address GSTIN. Used by both the preview and the save. |
| `src/modules/proforma-invoice/proforma-invoice.routes.ts`, `.controller.ts`, `.schema.ts` | New `POST /proforma-invoices/calculate` (nothing saved). Create and update accept the discount and line values, and the backend works out the tax. |
| `src/modules/proforma-invoice/proforma-invoice.repository.ts`, `.service.ts` | Saves the per-line HSN, rate, values and tax and the header split. PI create now runs in one transaction so a failure while saving the lines leaves the old PI as it was. |
| `src/modules/proforma-invoice/pi-change-request.service.ts` | Small change to pass the transaction through. |
| `src/modules/final-invoice/final-invoice.service.ts`, `.repository.ts` | The Final Invoice copies each line's HSN and rate from the PI line. It no longer reads the company's "active" Tax Profile. The tax is worked out by the same shared code. |
| `src/modules/final-invoice/final-invoice.routes.ts`, `.controller.ts`, `.schema.ts`, `.types.ts` | New `POST /final-invoices/preview`, so the create panel shows the same numbers that get saved. |
| `src/modules/master/item/item.service.ts`, `.repository.ts`, `.schema.ts`, `.types.ts` | Items carry `taxProfileCode`. A finished good on Sales must have a valid HSN and an active Tax Profile with a real GST percentage. A raw material cannot be on Sales (`ITEM_RAW_NOT_SALES`). |
| `src/modules/master/tax-profile/tax-profile.service.ts`, `.repository.ts` | A Tax Profile that items use cannot be switched off, deleted, or given a non-standard percentage (`TAX_PROFILE_IN_USE`, `TAX_PROFILE_RATE_INVALID`). |
| `src/shared/gst/gst-invoice-snapshot.ts`, `gst-invoice-source.repository.ts` | The e-Invoice and e-Way Bill use each line's own HSN and rate. On a per-item invoice the order charges and discount are already inside the lines, so there are no separate charge lines. |
| `src/shared/gst/gst-codes.util.ts` | Removed `DEFAULT_HSN_CODE`. There is no default HSN any more. |
| `src/modules/notifications/attachments/pi-sheet/*` (`pi-sheet.builder.ts`, `pi-sheet.loader.ts`, `pi-sheet.pdf.ts`, `pi-sheet.types.ts`, `final-invoice-sheet.loader.ts`) | The emailed PI and Final Invoice PDFs show the rate on each line and a tax summary per HSN and rate (PI) or tax rows per rate (Final Invoice), from the saved figures. |
| `src/modules/superadmin/superadmin.repository.ts`, `superadmin.service.ts` | When a new company is cloned from the template, the product items now carry the source item's HSN and Tax Profile instead of a fixed HSN. The clone also copies glass colours and shifts, copies product rates with their glass type and thickness, and uses `ON CONFLICT DO NOTHING` so it can be re-run. |
| `src/database/backfill-final-invoice-gst.ts` (removed), `package.json` | Removed the one-off script that filled the tax figures of Final Invoices from before migration 064, and its `backfill:final-invoice-gst` npm script. |

## Why
The old way could not tax an invoice that had two items with two different GST rates, and it guessed when data was missing (18% and a default HSN). The Tax Profile screen already existed but did not drive any tax. The real work was moving the maths into one backend place so the form, the saved document, the PDF and the e-Invoice all show the same numbers.

## How
**One calculation**: line taxable value = glass amount + that item's own machining charges, plus its share of order charges minus its share of the discount. Line tax = taxable value x the line's rate, rounded to 2 decimals. Same state gives half CGST and half SGST. Other state gives IGST. Invoice tax is the sum of the line taxes. If every item has the same rate, the result is exactly what the old way gave.

**Same state or other state**: the buyer's state code is the first two digits of the GSTIN on the billing address. It is compared with the seller's state. This is one rule used everywhere, replacing the Tamil Nadu only checks.

**Saved figures are final**: each document stores its own HSN, rate, taxable value and tax. Changing a Tax Profile later does not change a saved PI or Final Invoice. `tax_method = 'PER_LINE'` marks a new style document. NULL means an older document, which is read the old way.

**Raw material rule**: a raw material is bought and processed, so it is never on the Sales Account. It is enforced in the item service, in the tax lookup (finished goods on Sales only) and in the database check. A raw material and a finished good may share the same product, glass type, colour and thickness, because the item type is part of the unique index.

**Kept the same on purpose**: the old single-rate path is untouched for documents with `tax_method` NULL.

## Decisions made
**Preview endpoints instead of the browser calculating.** The create PI form and the create Final Invoice panel ask `/calculate` and `/preview`. I did not keep a copy of the formula in the browser, because two copies drift apart (this is what caused the earlier mismatch between the PI sheet and the view modal).

**Stop instead of falling back.** A missing HSN or Tax Profile returns a 400 that lists every item to fix. I did not keep the 18% or `70071900` fallback, because a silent wrong rate on a tax document is worse than a blocked save.

**No guessing between two matching items.** If two items match a line but have a different HSN or Tax Profile, it stops and asks for a colour on the order item that picks one.

**The sales check is `NOT VALID`.** The database check that a Sales finished good needs an HSN and Tax Profile only applies to new and edited rows. Old items that are not filled in do not block the migration. They are refused only when someone edits them, until they are filled.

**Three migrations became one.** The raw material and sales checks were first separate files (135 and 136). They were merged into 134 before it reached live. Everything in the file is safe to run twice. If another database already ran the old 134 on its own, it would miss the raw material and sales checks.

## Bugs found while testing (already there before, not fixed here)
**1. IGST shown for a Trichy address.** The buyer address had the dummy GSTIN `22AAAAA0000A1Z5` (state code 22, Chhattisgarh), so the tax correctly followed the GSTIN and applied IGST. This is bad data, not a code bug. Anyone billing to that address is affected. The GSTIN on the address has to be corrected.

**2. Six items have HSN 7305 or 7306.** These look wrong for glass. They were not changed here.

**3. Full backend test run has 6 failing tests in files this PR does not touch.** They are the purchase schema ("Item is required") and the pi-reminders slab tests.

## Test plan
**Automated tests (no new tests added):**
- [x] Backend type check (`npx tsc --noEmit`): passes.
- [ ] Tests for `computeInvoiceTax` and `resolveLineTax` - not written yet, see Notes.
- [ ] Full backend test run - 6 failures in files this PR does not touch (see "Bugs found while testing").

**Manual checks (local):**
- [x] Migration 134 applied on the local database (`rubine_dev`) and the item data was read back with read only queries.
- [ ] PI with one rate and with two rates on live data - not done, needs live data.
- [ ] Final Invoice made from a two rate PI - not done.
- [ ] e-Invoice in the sandbox with two rates - not done.
- [ ] PI and Final Invoice PDF and emailed PDF - not done.
- [ ] Migration 134 on live - not done, run by the team.
