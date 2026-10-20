# 2026-10-16 - final-invoice-sheet-and-pi-charges (frontend)

## Summary
This PR covers all the uncommitted frontend changes in the repo. The main part is the Final Invoice sheet: it did not look like the original tax invoice, and when the detail request failed the preview quietly showed an empty sheet with no products. The sheet is restyled to match the original, uses the real ISI image, shows polish quantity under SQM, and shows an error with a retry instead of an empty sheet. The PI Status drawer now has a per product Charges Breakdown, the Create Final Invoice panel shows PI Amount and PI Advance, and every row action menu (⋮) has a new compact look with tinted icons. "Replica" is renamed to "Clone" in the Order Booking list and on mobile.

No amounts, tax logic, API calls or other templates were changed on the invoice sheet. A type check shows no errors in the changed files. I did not open any screen in a browser, print preview or PDF myself, so every visual check below is for the reviewer to confirm.

**Insurance and Admin Charges still print as product rows on the invoice sheet** - see "Next steps" below. This is a pending choice, not a miss.

## What changed
| File | Change |
|---|---|
| `src/features/e-invoice/components/einvoice-tax-invoice-sheet.tsx` | Sheet styling and layout only. Title underline removed. The summary box is now a full grid with borders on every row and a bigger bold GRAND TOTAL row. A zero round off no longer prints as `-0.00`. The old drawn `IsiMark` SVG is replaced by the real ISI image. Table header cells are centred vertically. The line after the SQM header cell is drawn. In the body rows the PCS and SQM cells share one column (the divider shows only in the header). In the TOTAL row the right borders of Description, Case Nos. and Rate are removed. The Note / Reference area is shorter (165px, was 200px). The footer is now three boxes (jurisdiction, `E & O.E.`, signature block) with the text centred. |
| `src/features/e-invoice/components/einvoice-sheet-data.ts` | The summary label is now `GRAND TOTAL` in both sheet builders. In `buildFinalInvoiceSheetData`, a charge row with a fractional quantity (polish) now goes under SQM with PER `SQM`. Whole number quantities stay under PCS with PER `PCS`. |
| `src/features/e-invoice/components/isi-mark.png` (new) | The ISI mark image supplied by the user, imported with `?inline` so it also works in the print iframe and the PDF capture. |
| `src/features/final-invoices/components/final-invoice-preview-modal.tsx` | The detail query always re-reads when the preview opens. If the detail did not load, an amber "The invoice items could not be loaded" panel with a "Try again" button is shown, and Print and Download are disabled. |
| `src/features/final-invoices/components/create-final-invoice-panel.tsx` | Two new read only fields under the party details: PI Amount and PI Advance, shown as currency when a PI exists and as a dash when not. |
| `src/features/pi-status/components/pi-status-drawer.tsx` | Each product now has a clickable "Product N Charges" bar under its last line, which opens a Charges Breakdown table (name, qty, rate, %, amount, subtotal). Several can be open at once. Charges are looked up by order booking item id from the order bookings. The "Additional charges" list now ignores the PI's per product charge rows (the line items already include them), and for older PIs that only stored a header total it shows the leftover as one "Order Charges" line. The "WORK ORDER NO:" label is now "Work Order No:" in a larger, non upper case style. |
| `src/features/pi-status/components/pi-status-table.tsx` | The row ⋮ menu uses `MenuIcon`, tighter item padding, and no minimum width. |
| `src/components/ui/row-actions-menu.tsx` | New exported `MenuIcon` component: a 12px icon on a 24px rounded square tinted by the icon colour. `RowActionsMenu` uses it and the item padding is tighter. |
| `src/features/enquiries/enquiry-row-menu.tsx` | Row menu uses `MenuIcon` and the tighter item padding. |
| `src/features/order-booking/work-order-row-menu.tsx` | Row menu uses `MenuIcon` and the tighter item padding. |
| `src/features/order-booking/header-detail-tab.tsx` | Row menu uses `MenuIcon` and the tighter item padding. The "Replica" action, its confirm dialog and its toasts now say "Clone" / "Cloning…" / "cloned successfully" / "Could not clone this booking." |
| `src/features/pwa/order-booking/mobile-order-detail.tsx` | Same Replica to Clone wording on the mobile order detail (menu item, confirm sheet, toasts). |

## Why
The brief was to match the Final Invoice preview to the original tax invoice, with the Grand Total and tax summary box as the top priority. While doing it, the user found the products were missing from the preview. That traced back to a failed detail request (see the backend PR). The preview hid the failure behind an empty sheet, so it now says so.

The PI drawer and Create Final Invoice panel changes go with the backend charges work, so the screens show the same charge figures the Final Invoice now uses. The menu restyle makes all the ⋮ menus look the same.

## How
**Invoice sheet is visual only**: the sheet changes are CSS and markup inside the `.einv-sheet` stylesheet. The same stylesheet feeds the preview, print and PDF, so all three change together. The e-invoice preview uses the same sheet, so it picks up these changes too.

**ISI image**: imported with `?inline`, which gives a data URI. A normal file URL would not load inside the print iframe or the PDF capture.

**Polish quantity**: polish is charged per area, so its quantity is something like 5.355, not a piece count. The sheet now treats a non whole quantity as SQM. The amount is not recalculated, it is still the stored charge amount.

**Failed load**: `fetchFinalInvoiceDetail` swallows errors and returns null. That behaviour was kept. The modal now checks for null after loading and shows the retry panel.

**PI drawer breakdown**: the PI's saved items carry no charge lines, so the drawer loads each order booking and builds a map from order booking item id to its charges. The bar and panel only show once the charges for that item have loaded. Units come from the same `chargeUnit` helper as the new PI form.

**Menus**: `MenuIcon` picks its background and border from the colour in the icon class (for example `text-cyan-600`). Unknown colours fall back to slate.

**Kept the same on purpose**: the invoice summary rows, tax lines, round off and totals still come from the invoice record. Only the printed label case and the `-0.00` display changed.

## Decisions made
**GRAND TOTAL is upper case in the data, not by CSS.** A global uppercase style would also change the other labels, and the brief said not to do that. The label string is changed in both builders and the highlight check uses the new string.

**Footer spelling "Authorised Signatory" kept.** The reference image spells it "Authorized". The app already used "Authorised", so it was left alone.

**No signature or stamp was added.** The signature image and the digital signature lines still show only when real signature data exists.

**Replica is renamed to Clone only in the screen text.** Internal names such as `replicaMutation` and the `replica` pending state were left alone to keep the change small.

## Bugs found while testing (already there before, not fixed here)
**1. The HSN / SRVC. column is blank on the Final Invoice.** The final invoice items have no HSN field, so `buildFinalInvoiceSheetData` leaves it empty. It needs a data source first.

**2. Payment Conditions is blank on the Final Invoice.** Same cause, there is no value to read for it.

## Next steps
- Decide whether Insurance and Admin Charges should print in the summary box (between BASIC AMOUNT and ASSESSABLE VALUE) instead of as product rows. It is a one pattern change in `SUMMARY_CHARGE`. The basic amount would drop by those two charges and the Grand Total would stay the same.
- Find a source for HSN and Payment Conditions on the Final Invoice.

## Test plan
**Automated tests (none added):**
- [x] Type check (`tsc --noEmit -p tsconfig.app.json`): no errors in the changed files. One existing error in `src/routes/work-orders.tsx` is not from this change.
- [ ] No unit tests exist for the sheet builders, the drawer or the menus, so none were run.

**Manual checks (local):**
- [ ] Open a Final Invoice preview, reload, and confirm the products, the summary box and GRAND TOTAL show. Not checked in a browser by me.
- [ ] Print preview and Download PDF: confirm the ISI image, column lines and footer look the same as on screen. Not checked.
- [ ] Open the e-invoice preview and confirm it still renders. Not checked.
- [ ] PI Status drawer: open a PI, expand a "Product N Charges" bar, and confirm the subtotal matches the line items. Not checked.
- [ ] Create Final Invoice panel: confirm PI Amount and PI Advance show. Not checked.
- [ ] Open the ⋮ menu on the Enquiries, Work Orders, Order Booking and PI Status lists and confirm the new icons and spacing. Not checked.
- [x] The invoice summary amounts and the polish row were checked by reading the code against the screenshots the user sent, not by a live run.
