# 2026-11-05 - order-status-report-and-ui-fixes (frontend)

## Summary

This PR is a set of fixes from one day's work, mostly on the Order Status Register report plus small UI fixes in Final Invoice, Order Booking, Purchase, Production Plan and Work Order stock selection.

The Order Status Register report had mixed column alignment, showed the state in the Location column, had a WO Status filter that is no longer wanted, left Order Notes blank when only the product descriptions were filled in, and did not show when a PI's amounts were shared by several Work Orders. Elsewhere: the Final Invoice party approval showed an internal running number instead of the FI number, the Read from photo modal flashed open for enquiries without a photo, the Purchase Details modal repeated the product list in its payment section, the Production Plan allowed the same stage twice for one product, the stock picker's Done button could be pressed with nothing picked, and the Order Booking details modal showed the job type twice.

Each change keeps the existing flows working. All changes were tested in the browser. Frontend and backend type checks pass, and the changed report query was also run read-only against the database.

**ISSUE-03 of the report audit (Process "free text") is not changed on purpose** - see "Decisions made" below. This was a choice, not a miss.

## What changed

| File | Change |
|---|---|
| `src/features/reports/order-status/order-status-report.tsx` | Number columns (Thickness, Holes, Cutouts, quantities, areas, PI amounts) right-aligned, text left-aligned, dates centered via a new `dateColumn` helper. Location shows the city only. WO Status filter commented out, along with its PDF filter line and options list. PI Number shows `(n WOs)` when the PI covers more than one Work Order; the Excel export keeps the bare PI number. |
| `src/features/reports/order-status/api.ts` | Added `piWorkOrders` to the row type. |
| `src/lib/pdf/report-table-pdf.ts` | New optional `centerAlignColumns` option. Other reports are not affected. |
| `src/features/final-invoices/components/approve-final-invoice-modal.tsx` | Party Approval header shows the FI number (`finalInvoiceCode`), falling back to the running number. |
| `src/features/final-invoices/final-invoices-page.tsx` | Approval toast shows the FI number. |
| `src/features/pwa/final-invoices/mobile-final-invoice-detail.tsx` | Same toast fix in the mobile app. |
| `src/features/ai-extraction/read-from-photo-button.tsx` | The auto-read panel now starts closed and opens only after the enquiry is confirmed to have a photo. |
| `src/features/inventory/components/purchase-view-dialog.tsx` | Stock Transfer & Payment Details replaced by two side-by-side cards like the Proforma Invoice details modal: Amount Summary and Payment Details. The old card is commented out. |
| `src/features/order-booking/entry-table.tsx` | Headers with a tooltip (Charge Per) now sit on the same line as the other headers. |
| `src/features/production-plan/production-plan-editor.tsx` | A stage already chosen for a work order's product is hidden from the other rows, and Save / Submit reject a duplicate. |
| `src/features/order-booking/work-order-stock-section.tsx` | Done button in the stock picker is disabled until some stock is taken. |
| `src/features/order-booking/order-details-modal.tsx` | Job type badge removed (commented out) from the product card header; the Job Type field inside the section stays. |

## Why

All changes came from review feedback on the screens and from the Identified Issues Log in `docs/order-status-register-report-audit.md`. Most were small UI fixes. The real work was in the report query: building Order Notes without repeated text, and counting a PI's Work Orders across all filters so a single filtered row still shows its PI is shared.

## How

**Report alignment**: each column spec already carried an alignment used by both the screen table and the PDF. Dates now use `center`, and the PDF helper got a matching `centerAlignColumns` option.

**Order Notes**: the four texts are put in an array, empty ones dropped, duplicates removed by lowercase text, and the rest joined with ` · ` in a fixed order.

**FI number**: uses `finalInvoiceCode || finalInvoiceNo`, the same rule the rest of the Final Invoice screens already use.

**Read from photo**: the photo check was already there; the modal now opens after the check instead of opening first and closing after.

**Production Plan stage**: a stage is "taken" when another line has the same work order, product and stage. Taken stages are filtered from the dropdown, and validation catches the case where a row's work order or product is changed afterwards.

**Removed UI**: the WO Status filter, the old Purchase payment card and the Order Booking header badge are commented out, not deleted, so they can be restored.

## Decisions made

**Dates centered, not left-aligned.** The request was to right-align numbers "except date fields". Dates are centered, following the project's table alignment rule.

**Phone, Work Order No and PI Number stay left-aligned.** They are identifiers, not amounts.

**Location filter dropdown still shows "City, State".** Only the column was changed. The filter and the Location Sales report share the same backend query.

**Duplicate stage rule is per work order and product, not per whole work order.** Different products of one work order each have their own route, so each can still be planned at the same stage. This also means one product's stage can no longer be split over two lines of the same plan.

**Read from photo opens a moment later for enquiries with a photo.** It now waits for the photo check instead of opening straight away.

**Stock picker can still be closed with the close button when nothing is picked.** Stock is then picked automatically, as before. Only Done is disabled.

**Order Notes includes product descriptions.** This follows the audit and the original spec. Because the descriptions are often glass names, the column is busier than before.

**ISSUE-03 not changed.** The booking item's `type` is picked from the Glass Type master dropdown, not typed freely, and the spec maps the Process column to it. The audit's claim was wrong.

## Bugs found while testing (already there before, not fixed here)

**1. PI Balance subtracts the agreed advance, not the advance actually received.** The report's balance takes the PI's "Advance" payment term and assumes it was paid. If the customer has not paid the advance, the balance looks lower than what is really owed. This was there before this PR and affects anyone reading the PI Balance Amount column.

**2. The audit document has other inaccuracies.** For example, its § 8 describes a different Balance formula than the code uses. This affects anyone using that document as a reference.

## Notes

- All changes were tested in the browser.
- The current data has no PI shared by more than one Work Order, so the `(n WOs)` label was not seen on real data.

## Next steps

- Decide whether PI Balance should use the advance actually received, and file a ticket if so.
- Decide whether the Location filter dropdown should also show cities only.
- Correct or retire the remaining inaccurate parts of the report audit document.

## Test plan

**Automated tests:**
- [x] Frontend type check (`npx tsc --noEmit -p tsconfig.app.json`): no errors.

**Manual checks (local):**
- [x] Order Status Register: column alignment on screen and in the PDF, Location shows the city only, WO Status filter gone, Order Notes filled in. Tested in the browser.
- [x] Final Invoice Party Approval: header and toast show the FI number. Tested in the browser.
- [x] Enquiry to Order Booking: no Read from photo flash without a photo; modal opens and reads when there is one. Tested in the browser.
- [x] Purchase Details modal: Amount Summary and Payment Details cards, no repeated product list. Tested in the browser.
- [x] Add Product size table: Charge Per header aligned. Tested in the browser.
- [x] Create Production Plan: a stage chosen for a work order's product is not offered again. Tested in the browser.
- [x] Work Order stock picker: Done disabled until stock is taken. Tested in the browser.
- [x] Order Booking details modal: job type shown once. Tested in the browser.
