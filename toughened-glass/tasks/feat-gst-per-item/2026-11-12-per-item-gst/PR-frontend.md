# 2026-11-12 - per-item-gst (frontend)

## Summary
The screens assumed one GST rate for a whole invoice, with a fallback of 18% in several places. This PR makes the screens show what the backend works out for per-item GST. The item dialog now asks for an HSN code and a Tax Profile. The create PI form and the create Final Invoice panel ask the backend for their tax figures. The view modal, sheet and PDF show the saved rate on each line and a tax summary per HSN and rate. Older PIs and Final Invoices still show the old single-rate way. This depends on the backend PR of the same name. Only the type check was run (passes). Nothing was checked in the browser, including the red `*` on the item dialog fields.

## What changed
| File | Change |
|---|---|
| `src/features/items/item-dialog.tsx` | HSN Code and Tax Profile (GST) fields. A red `*` shows on both when the item is a Finished Good on the Sales Account. A Raw Material cannot be ticked for Sales (the box is disabled and cleared when the type changes). Error messages use a smaller text size. |
| `src/features/items/item-view-modal.tsx`, `types.ts` | Show and carry the new `taxProfileCode`. |
| `src/lib/pi-totals.ts` | Added `piTaxFromApi` and `applySavedTax`. For a new style PI they read the figures the backend saved (assessable value, CGST/SGST/IGST, round off, grand total, per HSN and rate summary). For an older PI they return nothing and the old formula is used. |
| `src/features/pi-status/api.ts`, `types.ts` | Attach the saved tax figures to each PI record. |
| `src/features/invoices/create-proforma-invoice.tsx` | The form calls `/proforma-invoices/calculate` for the tax and shows that result. The local rate arithmetic is gone. The save sends the discount and line values and the backend works out the tax again. |
| `src/features/final-invoices/api.ts` | New `previewFinalInvoice` call and `FinalInvoicePreview` type. `taxRate` can now be null (a PI with a rate on every line). |
| `src/features/final-invoices/components/create-final-invoice-panel.tsx` | Shows the tax, round off, total and due from the backend preview instead of its own maths. |
| `src/features/pi-status/components/pi-status-drawer.tsx` | The view modal shows the saved rate on each line and the tax summary. |
| `src/features/pi-status/components/proforma-invoice-sheet.tsx`, `proforma-pdf-data.ts`, `src/lib/pdf/proforma-invoice-pdf.ts` | The sheet and the downloaded PDF show the rate on each line and a summary per HSN and rate, from the saved figures. |
| `src/features/pi-status/components/approve-party-modal.tsx`, `src/features/pwa/pi-status/pi-approval-sheet.tsx` | Removed the guess "booking amount plus 18%". The total is the one the backend saved. |
| `src/features/pi-status/components/final-invoice-as-proforma-data.ts` | Builds the sheet data from the saved Final Invoice figures. |

## Why
With one rate for the whole invoice, a PI with two different GST rates could not be shown correctly, and the 18% fallbacks could show a wrong total. The backend now owns the tax maths, so the screens must stop recomputing it and just show the saved or previewed numbers.

## How
**Show, do not recompute**: for a new style PI the screens read the figures the backend saved, so changing an item's rate later does not change a saved document.

**Old documents**: `piTaxFromApi` returns nothing for a PI without the per-line marker, so those still go through the old formula and look the same as before.

**Required markers follow the backend rule**: the red `*` shows on HSN and Tax Profile only for a Finished Good with Sales ticked, which is the same case the backend refuses. It uses `useWatch` so it updates as the type and the Sales box change.

## Decisions made
**Preview from the backend, not a copy of the formula.** The form waits for `/calculate` and `/preview`. I did not keep a local formula as a fallback, so the screen can never show a number the backend will not save.

**Removed the 18% fallback in the approval screens.** If a PI has no saved total, the screen shows 0 instead of guessing from the booking amount.

## Notes
- The red `*` was first written with `form.watch`, then changed to `useWatch` after it did not show on the Edit Item dialog. The change has not been checked in the browser, so do a hard refresh and confirm it shows.
- Run the backend migration 134 before using these screens, otherwise the Tax Profile field has nothing to save into.

## Next steps
- Check the item dialog, the create PI form, the create Final Invoice panel, the view modal and the PDF in the browser, with a one rate PI and a two rate PI.

## Test plan
**Automated tests (no new tests added):**
- [x] Frontend type check (`npx tsc --noEmit -p tsconfig.app.json`): passes.

**Manual checks (local):**
- [ ] Item dialog: red `*` on HSN and Tax Profile for a Finished Good with Sales ticked, and the raw material Sales box locked - not done in the browser.
- [ ] Create PI with one rate and with two rates, and the numbers match the saved PI - not done.
- [ ] Create Final Invoice from a two rate PI, preview matches the saved invoice - not done.
- [ ] PI sheet, downloaded PDF and emailed PDF show the same numbers - not done.
- [ ] An older PI and Final Invoice still look the same as before - not done.
