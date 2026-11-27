# 2026-10-12 - work-order-print-and-scan (frontend)

## Summary

This PR carries on from the 2026-10-10 work-order-stages-and-pi-dialog PR. The printed Work Order and the Review sheet were missing data that exists in Order Booking (the logo note), showed hole and cutout columns in a different order from Order Booking, and always showed a blank Produced Quantity. This PR fixes those, adds a Scan Barcode button that is available on every page, shows the real barcode on the stage start panel, and changes the Approve button so that a work order without a delivery date or dispatch type goes to the Edit dialog first.

None of this was checked in a browser or against a printed PDF. I ran the TypeScript check after each change. See "Test plan".

**The PDF fonts are not changed on purpose** - see "Decisions made" below. This was a choice, not a miss.

## What changed

| File | Change |
|---|---|
| `src/lib/pdf/work-order-pdf.ts` | The header is two rows instead of three. Ref is removed and Dispatch Type moved up into the second row, between Proforma Inv No and Project. The unexplained "FT" is removed from the product banner. The banner now shows `Logo : <name>   Note : <note>`. Each table row has a `thickness`, and `producedQuantity` is one text value instead of one value per stage. |
| `src/lib/pdf/work-order-columns.ts` | Shared by the PDF and the Review sheet. The hole and cutout columns now follow Order Booking's size table: Hole, Centre Hole, Big Hole, Big Centre Hole, Cutout, Big Cutout. A new Thickness (mm) column sits after WO Sr. Glass Code and Spl Remark are removed. The seven stage columns (Cut, Grn, Fab, Deg, Tem, Lam, Ins) are replaced by one Produced Qty column with a total. |
| `src/features/order-booking/work-order-pdf-mapping.ts` | Fills the logo and note from the booking item. Produced Qty counts the finished pieces for each size row. Thickness is filled for each row. Glass code and special remark are no longer built. |
| `src/features/order-booking/work-order-sheet-view.tsx` | The Review sheet shows the logo and note on the route line. |
| `src/features/order-booking/api.ts` | `WorkOrderPiece` has two new fields from the backend: `orderBookingSizeId` and `completedStageCodes`. |
| `src/features/order-booking/work-order-summary-modal.tsx` | The status badge in the popup header is green when the work order is completed. In progress is amber, cancelled is grey. |
| `src/features/production-scan/work-order-pieces-table.tsx` | The product badge follows its pieces: Completed when every live piece is done, In Progress once one has started. An Approved or Denied review result still shows as it is. |
| `src/features/production-scan/components/barcode-image.tsx` (new) | Draws the real, scannable Code 128 barcode from a piece's barcode value. |
| `src/features/production-scan/components/stage-work-panel.tsx` | The "Approve / Start Stage" panel shows the piece's real barcode instead of the decorative picture. |
| `src/features/production-scan/scan-widget.tsx` (new) | A round Scan button fixed at the bottom right. It opens the existing scan dialog, and a scanned piece opens its own page. Only shown to users who can edit stages. |
| `src/components/layout/app-layout.tsx` | Renders the scan widget on every page. |
| `src/routes/work-orders.tsx` | The Approve icon (and the Approve button in the work order popup) checks the delivery date and dispatch type. If either is missing it opens the Edit dialog first. After saving there, the Review and Approve preview opens by itself. If both are filled, it goes straight to the preview. |
| `src/features/order-booking/work-order-edit-dialog.tsx` | New `onSaved` callback, called after a plain Save and before the dialog closes. |
| `src/features/pi-status/components/print-options-modal.tsx` | The sample row used for PI Status printing matches the new row shape. |

## Why

The printed sheet is what the shop floor reads, so it has to carry the same details as Order Booking and show real progress. The scan button was only on the Work Orders page, which made it slow to reach a piece from anywhere else. Approving a work order with no delivery date or dispatch type only showed a warning and a disabled button, so the user had to find the Edit action separately.

## How

**Produced Qty**: for each size row, the count of live pieces cut from that size whose status is completed. It goes up by one each time a piece is finished. A row with 5 pieces shows 0 until the first piece is done.

**Hole and cutout columns**: they use the same names and order as Order Booking, and map to the same size fields (`holeQty`, `centreHoleQty`, `bigHoleQty`, `bigCentreHoleQty`, `cutoutQty`, `bigCutoutQty`).

**Approve flow**: the page remembers which work order is waiting to be approved. When the Edit dialog saves that work order, the page opens the approval preview. Closing the dialog without saving clears it, so nothing opens by itself.

**Scan widget**: it reuses `ScanDialog` as it is, so the scan, mismatch and camera behaviour is unchanged.

## Decisions made

**The Work Order PDF still uses `helvetica`.** It is a generated file, so it cannot use the web font without embedding a font file.

**Produced Qty counts finished pieces, not pieces per stage.** You asked for one column without the stages. A piece counts only when every production stage is done.

**The scan button in the Work Orders page header was kept.** The new floating button is in addition to it.

**"FT" was removed without finding out what it stands for.** If it meant something such as "fully tempered", it can be put back.

## Bugs found while testing (already there before, not fixed here)

**1. Deg, Lam and Ins stages are no longer shown at all.** They were blank before because no route used them. With one Produced Qty column this no longer matters.

**2. A work order with a past delivery date can still be saved if the date was already stored.** The date picker blocks new past dates, but an old stored date is not changed.

## Notes

- Several of these changes were already committed or merged when this document was written, so the file list comes from the work done in this task, not from a fresh diff.
- Produced Qty depends on the backend change in `PR-backend.md` (each piece now carries its size row id). The backend needs a restart first, or the column shows 0.
- `completedStageCodes` is returned by the backend but not used by the frontend any more, since Produced Qty is no longer per stage.

## Next steps

- Embed the app font in the PDFs if the printed pages should match the screens.
- Remove `completedStageCodes` from the backend if nothing else needs it.
- Add a shared header component for the stages, piece and booking pages.

## Test plan

**Automated tests (no test files for these screens):**
- [x] `npx tsc --noEmit -p tsconfig.app.json`: passed after the last change, with no errors.

**Manual checks (local):**
- [ ] Print a completed work order: Ref is gone, Dispatch Type is in the second row, no "FT", the logo and note show, columns are in the Order Booking order, and Produced Qty shows the finished count. Not checked against a real PDF.
- [ ] Review sheet: Thickness column after WO Sr, no Glass Code or Spl Remark, logo and note on the route line. Not checked in a browser.
- [ ] Complete one piece of a product with several pieces and check that Produced Qty goes to 1. Not checked.
- [ ] Work order popup: the status badge is green when completed. Not checked in a browser.
- [ ] Pieces table: the product badge shows Completed when all pieces are done. Not checked in a browser.
- [ ] Stage start panel: the real barcode is shown and a scanner can read it. Not checked.
- [ ] Scan widget: the button shows on other pages and opens the scan dialog. Not checked.
- [ ] Approve on a work order with no delivery date: the Edit dialog opens, and after Save the approval preview opens. Not checked.
