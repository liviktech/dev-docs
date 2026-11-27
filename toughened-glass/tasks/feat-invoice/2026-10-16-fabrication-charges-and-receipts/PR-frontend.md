# 2026-10-16 - fabrication-charges-and-receipts (frontend)

## Summary
The Create Final Invoice panel and the PI Status detail did not show item fabrication charges, so the totals did not match the backend. The Final Invoices table mixed the payment receipt into the Photos window, and the Photos window on PI Status showed PI receipts that already have their own button. Confirm buttons across several dialogs were also inconsistent. This PR shows fabrication charges in both places, adds a separate "Payment Slip / Receipt" column to Final Invoices, hides PI receipts from the PI Status photo list, and tidies the dialog buttons.

Both repos compile clean with a type check. Nothing was checked in the running app and no automated tests exist for these screens.

## What changed
| File | Change |
|---|---|
| `src/features/final-invoices/components/create-final-invoice-panel.tsx` | Item rows show glass plus fabrication. New "Fabrication Charges" line. Proration now uses glass value only. Order level charges come from `piOrderCharges`. |
| `src/features/final-invoices/api.ts` | New optional `fabricationAmount`, `totalAmount` and `piOrderCharges` fields. |
| `src/features/final-invoices/components/final-invoices-table.tsx` | Photos column replaced by a new "Payment Slip / Receipt" column with a View Receipt button (new `PaymentReceiptButton`). |
| `src/features/final-invoices/final-invoice-as-proforma-data.ts` | Handling charge is 0 when there is a charges list. Adds advance amount, advance percent and balance amount to the sheet totals. |
| `src/features/pi-status/components/pi-status-drawer.tsx` | Line items carry the fabrication amount. Glass total, fabrication total and basic amount now sum the line items first. Photos button hides PI receipts. |
| `src/features/pi-status/components/pi-status-table.tsx` | View Receipt button restyled to match. Photos column hides PI receipts. |
| `src/features/pi-status/pi-status-page.tsx` | Delete dialog now shows the standard warning line instead of the long description. |
| `src/features/ai-extraction/photo-review-panel.tsx` | New `hidePIReceipts` prop. New `withOrderPhotos` mode shows all order photos plus the extras. Titles and empty messages follow the new mode. |
| `src/features/ai-extraction/types.ts` | New optional `withOrderPhotos` on `ExtraPhotosSource`. |
| `src/features/ai-extraction/photos-button.tsx`, `photos-column.tsx` | Pass `hidePIReceipts` through. |
| `src/components/ui/confirm-action-modal.tsx` | New optional `confirmButtonClassName`. |
| `src/features/enquiries/enquiries-page.tsx` | Approve button is now "Confirm" in cyan, Convert is cyan. |
| `src/features/dispatch/components/approve-dispatch-modal.tsx`, `src/features/e-way-bill/components/generate-ewaybill-modal.tsx` | Smaller buttons, cyan 700 colour, pointer cursor. The submit button reads "Confirm". |
| `src/routes/work-orders.tsx` | Reject dialog adds a line saying the work order moves to Rejected. |

## Why
The backend now sends fabrication charges per item, and the screens needed to show them so what the user sees adds up. The receipt and dialog changes came from UI consistency feedback on the same screens.

## How
**Fabrication in the panel**: each item row shows glass plus fabrication. "Total Material" is glass only, "Fabrication Charges" is a separate line, and "Total Amount" is both together. Proration uses glass value only, matching the server.

**PI detail sums**: totals sum the line items first and fall back to the order group totals and then the stored basic amount.

**Receipt column**: the button opens `PhotoReviewPanel` with only the receipt, loaded when the window opens because it is a large data URI.

**Photo list modes**: `extraPhotos` alone still shows only the extras (PI Status View Receipt). With `withOrderPhotos` it shows the order photos and then the extras. `hidePIReceipts` removes PI receipts from the order photo list.

**Kept the same on purpose**: the backend still decides the real amounts at creation. The panel numbers are a preview only.

## Decisions made
**The Final Invoices receipt is its own column.** The other option was to keep it inside the Photos window. A separate button matches PI Status and the receipt is no longer buried among order photos.

**Button labels now read "Confirm".** Approve and Generate buttons in the confirm dialogs now say "Confirm" for one consistent action word.

## Bugs found while testing (already there before, not fixed here)
**1. `final-invoice-as-proforma-data.ts` reads the record through `as any`.** The new advance and balance fields use `(record as any)` with snake case fallbacks, the same loose style the file already used. The record type should hold these fields so a type check can catch a rename.

## Notes
- `withOrderPhotos` is added to the types and the panel but no screen sets it in this diff, so that mode is unused for now.
- The Final Invoices Photos column is gone. Order photos are no longer opened from that table.

## Next steps
- Type the advance and balance fields on the Final Invoice record and drop the `as any`.
- Use `withOrderPhotos` where Final Invoice photos need the order photos too, or remove it.

## Test plan
**Automated tests (none added):**
- [ ] No test file covers the Create Final Invoice panel, the PI drawer or the receipt button - not added in this PR.
- [x] Frontend type check (`npx tsc -b --noEmit`): no errors.

**Manual checks (local):**
- [ ] Open Create Final Invoice for a PI with fabrication charges and compare totals with the backend - not run, the app was not started.
- [ ] Click View Receipt on a Final Invoice row and on a PI Status row - not run, same reason.
- [ ] Check the Confirm buttons and the new dialog text on enquiries, dispatch, e-Way Bill and work orders - not run, same reason.
