# 2026-10-20 - downtime-progress-and-delivery-updates (frontend)

## Summary
This PR adds the screens for a batch of production and dispatch changes. The stage page gets a Downtime button and dialog. Work Orders and PI Status get a Progress column that opens a per product, per stage breakdown. Enquiries and final invoices get a delivery date. The barcode label says WOD instead of MFD. Delivery type becomes Loose packing or Wooden packing. The final invoice PDF and Print buttons only show after admin approval. Dispatch no longer asks for a vehicle type. The Dispatch page can update the vehicle on a live e-Way Bill with a remark, and the update vehicle modal now matches the Approve Dispatch modal buttons.

`npm run build` succeeds. The screens were not clicked through by me for this PR, see "Test plan".

## What changed
| File | Change |
|---|---|
| `src/features/production-scan/components/downtime-dialog.tsx` (new) | The Downtime dialog for the current stage: list the stage's downtimes, add one (start, optional end, reason) and fill in an end time later. Start and End are stacked, each using the existing `DatePicker` (same calendar as the rest of the site) plus typed hour and minute boxes and an AM / PM toggle (12 hour time). End has a Clear link. The value stays `YYYY-MM-DDTHH:mm` in 24 hour form, so nothing changes for the API. No new shared component was added. |
| `src/features/production-scan/components/work-order-progress.tsx` (new) | The Progress cell (single line bar, stages done and percent, clickable look with a chevron) and the dialog that opens from it with each product's stages, pieces done, bar and status. Used by both Work Orders and PI Status. |
| `src/features/production-scan/scan-detail-page.tsx` | Added the Downtime button (shows a red dot when a downtime is still open) and mounts the dialog. |
| `src/features/production-scan/components/stage-work-panel.tsx` | The Complete button is disabled, with a tooltip, while the stage has a downtime with no end time. |
| `src/features/production-scan/api.ts` | Added `fetchWorkOrderStageProgress`, `createDowntime` and `updateDowntime`. |
| `src/features/production-scan/queries.ts` | Added `workOrderStageProgressQueryOptions` (refetches every 20 seconds) and the create and update downtime mutations. |
| `src/features/production-scan/types.ts` | Added the downtime type, the `downtimes` list on a stage, and the progress types. |
| `src/routes/work-orders.tsx` | The Progress column now renders `WorkOrderProgress` instead of the old inline bar with a hover title. |
| `src/features/pi-status/components/pi-status-table.tsx` | Added a Progress column using the same scan list data and `WorkOrderProgress`. Shows a dash when the PI has no released work order. |
| `src/features/enquiries/api.ts` | Added `deliveryDate` to the enquiry types. |
| `src/features/enquiries/enquiry-form-modal.tsx` | Added a Delivery Date date picker (not earlier than the enquiry date), loaded from and saved with the enquiry. |
| `src/features/pwa/enquiry/mobile-enquiry-form.tsx` | Added the Delivery date field to the mobile enquiry form. |
| `src/features/final-invoices/api.ts` | Maps `delivery_date` and sends `deliveryDate` on create. |
| `src/features/final-invoices/types.ts` | Added `deliveryDate` and `delivery_date`. |
| `src/features/final-invoices/components/create-final-invoice-panel.tsx` | Added a Delivery Date picker (today or later) and sends it on create. |
| `src/features/final-invoices/components/final-invoice-detail-modal.tsx` | Shows the Delivery Date. |
| `src/features/final-invoices/components/final-invoice-preview-modal.tsx` | The PDF download and Print buttons only show when the invoice has an admin approval date. |
| `src/lib/pdf/barcode-pdf.ts` | The date on the label reads `WOD` instead of `MFD`. |
| `src/lib/delivery-type.ts` (new) | The two delivery type options, a type guard and `deliveryTypeLabel` (unknown old values show as they are). |
| `src/features/order-booking/schema.ts` | The work order form accepts `LOOSE_PACKING` and `WOODEN_PACKING`. |
| `src/features/order-booking/api.ts` | `UpdateWorkOrderBody.dispatchType` uses the new values. |
| `src/features/order-booking/work-order-form-fields.tsx` | Options and label changed to Delivery Type. |
| `src/features/order-booking/work-order-tab.tsx` | Options and label changed to Delivery Type. |
| `src/features/order-booking/work-order-edit-dialog.tsx` | Maps the new values to and from the form. |
| `src/features/order-booking/factory-approval-sheet-dialog.tsx` | The missing field message says Delivery Type. |
| `src/features/order-booking/work-order-sheet.tsx` | The printed sheet label says Delivery Type and uses `deliveryTypeLabel`. |
| `src/features/order-booking/work-order-sheet-view.tsx` | Same label and value change on the on screen sheet. |
| `src/features/order-booking/work-order-summary-modal.tsx` | Same label and value change in the summary. |
| `src/lib/pdf/work-order-pdf.ts` | Same label and value change in the work order PDF. |
| `src/features/pi-status/components/print-options-modal.tsx` | The sample work order data uses `LOOSE_PACKING`. |
| `src/features/pwa/work-orders/mobile-work-order-detail.tsx` | Delivery Type label, new options, default `LOOSE_PACKING`. |
| `src/features/dispatch/types.ts` | Removed `VehicleType` and the vehicle type fields. Added `remarks`. |
| `src/features/dispatch/api.ts` | Stopped mapping and sending the vehicle type. Maps `remarks`. |
| `src/features/dispatch/queries.ts` | The approve mutation no longer takes a vehicle type. |
| `src/features/dispatch/components/approve-dispatch-modal.tsx` | Removed the Vehicle Type toggle. Confirm gets the vehicle number and driver only. |
| `src/features/dispatch/components/dispatch-detail-modal.tsx` | Removed the Vehicle Type field. |
| `src/features/dispatch/components/dispatch-table.tsx` | Removed the Vehicle Type column. Added an Update Vehicle row action, shown only when the invoice has a live e-Way Bill and the user is allowed. |
| `src/features/dispatch/components/gate-pass-sheet.tsx` | Removed the Vehicle Type cell. |
| `src/features/dispatch/dispatch-page.tsx` | Added the Update Vehicle flow: opens the modal, calls the update, shows a success toast. Needs EDIT on `e_way_bill`. |
| `src/features/e-way-bill/components/update-ewaybill-vehicle-modal.tsx` | Added a Remarks (reason for change) box. Buttons match the Approve Dispatch modal and "Update" is now "Confirm". |
| `src/features/e-way-bill/queries.ts` | The vehicle update also refreshes the Dispatch list. Added `useUpdateEwayBillVehicleForInvoiceMutation` for the Dispatch page. |
| `src/features/e-way-bill/types.ts` | Added `remarks` to `UpdateVehicleInput`. |
| `src/features/e-invoice/types.ts` | The dispatch snapshot no longer has `vehicleType`. |
| `src/features/e-invoice/components/einvoice-sheet-data.ts` | Removed `transporterName` from the transport data and its type. It was only ever filled from the vehicle type, so it would always be blank. |
| `src/features/e-invoice/components/einvoice-tax-invoice-sheet.tsx` | Removed the Transporter Name row from the transport block. Vehicle No., Way Bill no, Destination, Removal Date and Driver Name are unchanged. |
| `src/features/pwa/dispatch/dispatch-approval-sheet.tsx` | Removed the Vehicle Type buttons. Confirm gets vehicle number and driver only. |
| `src/features/pwa/dispatch/dispatch-format.ts` | Removed `vehicleTypeLabel`. |
| `src/features/pwa/dispatch/mobile-dispatch-detail.tsx` | Removed the Vehicle Type row and the type argument on approve. |
| `src/features/pwa/dispatch/mobile-dispatch-list.tsx` | The card shows `Vehicle <number>` instead of the type and number. |
| `src/features/invoices/create-proforma-invoice.tsx` | Removed an unused `totalActualSqm` value. No behaviour change. |
| `src/features/pi-status/components/pi-status-drawer.tsx` | Removed an unused `totalActualSqm` value. No behaviour change. |

## Why
The shop floor needs to record machine breakdowns, and to be stopped from completing a stage while one is open. Managers wanted to see how far each work order is, per stage, from the Work Orders and PI Status lists without opening each piece. Sales and accounts needed delivery dates on enquiries and final invoices. The label and delivery type wording was changed to match how the business talks about these. Unapproved final invoices should not be downloadable or printable. Dispatch no longer needs a vehicle type, and a vehicle change on a live e-Way Bill needed a reason and a way to do it from the Dispatch page. This depends on the backend PR of the same name.

## How
**Progress cell**: One compact line with "x/y stages", the percentage, a bar and a chevron, with a cyan border so it looks clickable. Clicking opens the dialog, which loads `/production-scan/work-orders/:id/progress`. The hover tooltip was removed on purpose. The same component is used in both tables, so they stay identical. PI Status matches a PI to progress by `workOrderId`, which the backend list query now returns.

**Downtime**: The button sits next to the other piece actions and is disabled when the user cannot scan. The dialog edits the stage the piece is currently at. A red dot on the button shows an open downtime, and the Complete button waits for its end time.

**Delivery type**: All lists, forms, sheets and PDFs read from one helper, `src/lib/delivery-type.ts`. The field is still called `dispatchType` in the data. Only the visible wording and the allowed values changed. Old stored values are shown as they are.

**Admin approval gate**: The final invoice preview checks `adminApprovedAt` and hides the PDF and Print buttons until it is set. The preview itself is still visible.

**Update vehicle from Dispatch**: The modal needs an e-Way Bill record, but the Dispatch page only has the dispatch row. The page builds a minimal record from the row (number and current vehicle), and the real update goes by final invoice id.

## Decisions made
**Rejected pieces are not listed in the progress dialog.** A "rejected" count is shown next to the pieces done text, but no list of which pieces were rejected, and no per stage badge or note. Stage rows only count live pieces, so they can show 100% even when pieces were rejected at that stage. This was a choice, not a miss.

**Kept the old Dispatch Type field name in code.** Renaming the property would touch the API and the database for no user visible gain.

**Transporter Name row removed from the e-invoice sheet.** It used to be filled from the vehicle type ("OWN VEHICLE" or "COMPANY VEHICLE"). With no type there is nothing to fill it from, so it would always print blank. We removed the row instead of leaving an empty one, and did not add a transporter field on the dispatch. Checked first: the row is rendered in one place only, nothing else reads `transporterName`, and it is not sent in any API payload. The left column of the transport block now has one more row than the right, which was not checked visually.

## Notes
- The Dispatch page's Update Vehicle action only appears for invoices that have an active e-Way Bill.
- Old work orders with `REGULAR` or `EXPRESS` still display, but the edit form will ask for one of the new values.

## Test plan
**Automated tests (all passing):**
- [x] `npm run build`: build succeeded (before the Transporter Name removal).
- [x] `tsc --noEmit -p tsconfig.app.json`: no errors after the Transporter Name removal and the Downtime dialog date and time change.
- [ ] No frontend unit tests were added or run for these screens.

**Manual checks (local):**
- [ ] Downtime dialog, progress dialog, delivery date fields, the approval gated PDF buttons and the Update Vehicle flow were not clicked through by me in a browser for this PR. Please check them in the running app before merging.
- [x] The backend migrations these screens depend on (068 to 072) were applied locally.
