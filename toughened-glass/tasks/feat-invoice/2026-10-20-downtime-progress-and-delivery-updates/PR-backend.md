# 2026-10-20 - downtime-progress-and-delivery-updates (backend)

## Summary
This PR adds the backend side of a batch of production and dispatch changes. Stage downtime can now be recorded and edited, and a stage cannot be completed while a downtime has no end time. There is a new endpoint that returns per product, per stage progress for a work order. Enquiries get a delivery date, and final invoices get a delivery date entered at creation. Dispatch no longer stores a vehicle type, it stores remarks instead, and an e-Way Bill Part-B vehicle update now also updates the dispatch. The work order delivery type values change from Regular/Express to Loose packing/Wooden packing.

Migrations 068 to 072 were applied to the local database. The TypeScript check passes. The new endpoints were not called by hand in this PR, see "Test plan".

## What changed
| File | Change |
|---|---|
| `src/database/migrations/068_dispatch_drop_vehicle_type.sql` (new) | Drops the `chk_dispatch_vehicle_type` constraint and the `vehicle_type` column from `invoice.dispatch`. |
| `src/database/migrations/069_dispatch_remarks.sql` (new) | Adds a nullable `remarks` column to `invoice.dispatch`. |
| `src/database/migrations/070_final_invoice_delivery_date.sql` (new) | Adds a nullable `delivery_date` column to `invoice.final_invoice`. |
| `src/database/migrations/071_work_order_process_downtime.sql` (new) | Creates `production.work_order_process_downtime` (start time, optional end time, reason, created/updated by) with a check that the end is not before the start, plus indexes on process and company. |
| `src/database/migrations/072_enquiry_delivery_date.sql` (new) | Adds a nullable `delivery_date` column to `sales.enquiry`. |
| `src/modules/dispatch/dispatch.types.ts` | Removed the `VehicleType` type and `vehicle_type` from the dispatch, table and gate pass shapes. Added `remarks`. |
| `src/modules/dispatch/dispatch.schema.ts` | Removed `vehicleType` from `approveDispatchSchema`. |
| `src/modules/dispatch/dispatch.service.ts` | `approveDispatch` no longer passes a vehicle type. |
| `src/modules/dispatch/dispatch.repository.ts` | Removed `vehicle_type` from the table query, approve query (parameters renumbered) and gate pass select. The table query now returns `remarks`. Added `updateDispatchVehicleByInvoice`, which sets the vehicle number and remarks on the dispatch of a final invoice. |
| `src/modules/e-way-bill/e-way-bill.schema.ts` | `updateVehicleSchema` accepts an optional `remarks` (trimmed, max 500). |
| `src/modules/e-way-bill/e-way-bill.service.ts` | After a successful Part-B vehicle update, `updateVehicle` also calls `updateDispatchVehicleByInvoice` so the Dispatch table and Gate Pass show the new vehicle and the reason. |
| `src/modules/enquiry/enquiry.types.ts` | Added `delivery_date` to `EnquiryRow` and `deliveryDate` to `Enquiry`. |
| `src/modules/enquiry/enquiry.schema.ts` | `createEnquirySchema` accepts an optional, nullable `deliveryDate` (YYYY-MM-DD). |
| `src/modules/enquiry/enquiry.repository.ts` | Create inserts `delivery_date`. Update sets it when it is part of the payload. |
| `src/modules/enquiry/enquiry.service.ts` | `toEnquiry` maps `delivery_date` to `deliveryDate`. |
| `src/modules/final-invoice/final-invoice.types.ts` | Added `delivery_date` to `FinalInvoiceRow`. |
| `src/modules/final-invoice/final-invoice.schema.ts` | `createFinalInvoiceSchema` accepts an optional `deliveryDate` (YYYY-MM-DD). |
| `src/modules/final-invoice/final-invoice.service.ts` | Passes `deliveryDate` through to the repository. |
| `src/modules/final-invoice/final-invoice.repository.ts` | `createFinalInvoiceInTx` takes and inserts `delivery_date`. The list table select returns `fi.delivery_date`. |
| `src/modules/production-scan/production-scan.types.ts` | Added the progress shapes (`ScanStageProgress`, `ScanProductProgress`, `ScanWorkOrderProgress`) and the downtime shapes (`ScanDowntimeRow`, `ScanDowntime`). Each `ScanDetailStage` now carries `downtimes`. |
| `src/modules/production-scan/production-scan.schema.ts` | Added `downtimeParamSchema`, `createDowntimeSchema` and `updateDowntimeSchema` (valid date-times, reason required, max 500). |
| `src/modules/production-scan/production-scan.repository.ts` | Added `listDowntimesForItems`, `findProcessOfItem`, `insertDowntime`, `findDowntimeOfItem`, `updateDowntime` and `countOpenDowntimesForProcess`. |
| `src/modules/production-scan/production-scan.service.ts` | Piece detail now includes each stage's downtimes. `completeScanStage` refuses to complete a stage that has an open downtime. Added `createDowntime`, `updateDowntimeRecord` (end can not be before start) and `getWorkOrderStageProgress`. |
| `src/modules/production-scan/production-scan.controller.ts` | Added `createDowntime`, `updateDowntime` and `getWorkOrderStageProgress` handlers. |
| `src/modules/production-scan/production-scan.routes.ts` | Added `GET /work-orders/:id/progress`, `POST /:id/downtimes` and `PATCH /:id/downtimes/:downtimeId`. The scan list and the progress route now also allow the `pi_status` VIEW permission. |
| `src/modules/proforma-invoice/proforma-invoice.repository.ts` | The PI list query now also selects `pi.work_order_id`, so the PI Status page can match each PI to its work order's progress. |
| `src/modules/work-order/work-order.schema.ts` | `dispatchTypeSchema` is now `LOOSE_PACKING` or `WOODEN_PACKING`. |
| `src/shared/gst/gst-invoice-source.repository.ts` | Removed `vehicle_type` from the dispatch part of the GST invoice source (type and query). |

## Why
The shop floor needed a way to record machine breakdowns on a stage, and to stop a stage being completed while one is still open. Sales and accounts needed a delivery date on enquiries and final invoices. Dispatch no longer wants a vehicle type (Own or Company). When a vehicle is changed on a live e-Way Bill, the reason needed to be kept, and the Dispatch page and Gate Pass should show the same vehicle as the e-Way Bill. The Work Orders and PI Status tables needed a per stage progress view, which needed a new endpoint. The delivery type values changed to Loose packing and Wooden packing.

## How
**Downtime**: A downtime belongs to one stage (work order process) of one piece. Create takes the stage id, a start time, an optional end time and a reason. Update can fill in the end time later. Completing a stage counts open downtimes (end time empty) and throws `SCAN_STAGE_DOWNTIME_OPEN` if there are any. Both write routes need CONTROL on `barcode_scanner` or `work_order`.

**Progress endpoint**: `getWorkOrderStageProgress` groups the work order's pieces by product. Only live pieces of products that are not denied are counted (the same rule the Production Line list uses). For each stage it counts total, completed and in progress pieces, and it also returns the number of rejected pieces per product. System stages are left out, as in the list.

**Delivery date**: Stored as a plain DATE on the enquiry and on the final invoice, all optional, so existing rows are unaffected.

**Vehicle on dispatch**: Approving a dispatch now only saves the vehicle number and the driver. The e-Way Bill Part-B update copies the new vehicle number onto the dispatch and sets the remarks (an empty remark keeps the old one, via `COALESCE`). It only runs when a vehicle number was part of the update.

**Kept the same on purpose**: The stored delivery type values are not rewritten. The schema only accepts the two new values for new saves, and old values still display as they are on the frontend.

## Decisions made
**Dropped the vehicle type column instead of leaving it unused.** The type is gone from the UI, the API and the GST invoice source, so keeping a dead column would only confuse the next person. The cost is that existing vehicle type data is lost once migration 068 runs.

**Remarks live on the dispatch, not on the e-Way Bill.** The Dispatch table and Gate Pass read from the dispatch, so one place to read from was simpler than joining to the e-Way Bill.

**The progress endpoint is separate from the scan list.** The list only carries per work order totals. The per product, per stage numbers are only needed when someone opens the dialog, so they are loaded on demand.

## Notes
- Existing work orders may still hold the old `REGULAR` or `EXPRESS` value. They are not migrated here and the frontend shows them as they are. Saving such a work order again requires picking one of the new values.
- Migration 068 deletes the `vehicle_type` data in `invoice.dispatch`.

## Next steps
- Decide whether old `REGULAR` and `EXPRESS` work orders should be converted to one of the new values.

## Test plan
**Automated tests (all passing):**
- [x] `npx tsc --noEmit`: no TypeScript errors.
- [ ] No unit or integration tests were added for downtime, the progress endpoint or the vehicle sync. The repo has no tests for these modules and none were written here.

**Manual checks (local):**
- [x] Migrations 068 to 072 were run against the local database and applied.
- [ ] The new endpoints (`/production-scan/work-orders/:id/progress`, `/production-scan/:id/downtimes`) were not called directly with a REST client in this PR. They are exercised through the frontend screens in the frontend PR.
- [ ] The "cannot complete a stage with an open downtime" rule and the e-Way Bill to Dispatch vehicle sync were not run end to end against a real e-Way Bill provider.
