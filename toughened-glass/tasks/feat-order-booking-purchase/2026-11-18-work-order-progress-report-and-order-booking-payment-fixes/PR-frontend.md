# 2026-11-18 - work-order-progress-report-and-order-booking-payment-fixes (frontend)

## Summary

This PR is the second batch of work from one day (the first batch is in `docs/PR-department-daily-report-po-purchase-columns-and-fixes.md`). The main piece is a new **Work Order Progress** report. The rest is a set of fixes in Order Booking, the PI Admin Approval and the payment screens.

Before this PR there was no report showing how far one work order has progressed. The Photos button in the Work Orders and Order Booking tables was enabled on every row, even for orders with no photos. Reading a photo into the Add Product table skipped any glass the booking already had, so new sizes of the same glass were dropped. Color was optional in Add Product. The Work Order view showed the Lot number twice. Bank transfer and cheque payments had no transaction date. Admin Approval with a payment slip attached failed, because the promised delivery date was left out of the request.

Each change keeps the existing flows working. All changes were tested in the browser. Frontend and backend type checks pass. The new report's queries, the photo flags, the transaction date rules and the photo merge logic were also run on a scratch local Postgres or as plain functions.

**Moving the slip upload into the approval request on the server is not included on purpose** - see "Decisions made" below. This was a choice, not a miss.

## What changed

| File | Change |
|---|---|
| `src/features/reports/work-order-progress/api.ts` (new) | Row type and request for the new report. Nothing is requested until a work order is picked. |
| `src/features/reports/work-order-progress/labels.ts` (new) | Status names, number and date formatting for the report. |
| `src/features/reports/work-order-progress/work-order-progress-report.tsx` (new) | The report screen: work order dropdown, Reset, one standard table, Excel and PDF. The Stage Progress column draws a thin progress bar. |
| `src/features/reports/report-menu.ts`, `src/features/reports/registry.ts` | "Work Order Progress" added to the Production Plan group. |
| `src/features/pwa/reports/work-order-progress-report.tsx` (new), `src/features/pwa/reports/registry.ts` | The mobile version, on the standard mobile report list. |
| `src/features/order-booking/api.ts` | `isPhoto` added to the Order Booking table row and the Work Order list item types. |
| `src/features/order-booking/draft-products.ts` | A photo row is skipped as "already saved" only when its sizes are already on the booking, not just its glass name. |
| `src/features/order-booking/products-tab.tsx` | Passes the booking's saved sizes to the merge, runs the merge once (so the message shows once), and no longer keeps the list of saved sheet titles. |
| `src/features/order-booking/schema.ts`, `add-product-modal.tsx`, `product-detail-panel.tsx` | Color is required in Add Product. A new product opens with nothing chosen; "Colorless" is a valid choice. |
| `src/features/order-booking/work-order-summary-modal.tsx` | The Lot ID field in Work Order Details is commented out; the Lot No column in the Products table stays. |
| `src/features/payment-details/types.ts`, `payment-details-fields.tsx` | New Transaction Date for bank transfer and cheque, in the form used by PI Admin Approval and the Final Invoice payment (desktop and mobile). It also shows in the read-only payment view. |
| `src/features/inventory/components/purchase-payment-dialog.tsx`, `purchase-view-dialog.tsx`, `purchase-edit-requests-dialog.tsx` | "Payment Date" renamed "Transaction Date". |
| `src/features/pi-status/components/approve-party-modal.tsx` | The confirm step rebuilt: one typed `ApprovalConfirmation`, one `onConfirm` call per side, everything checked before the slip is uploaded, and the slip is uploaded once. |
| `src/features/pi-status/pi-status-page.tsx` | Reads the single `ApprovalConfirmation` the modal sends. |

## Why

The Work Order Progress report is a new reporting need: one place to see how far a work order is, stage by stage and product by product. The Order Booking, payment and approval changes came from review feedback and from bugs found while testing: the Photos button that never greyed out, photos with new sizes being thrown away, the double Lot number, and the Admin Approval failure.

Most of the work was in the report. The data already existed in the production tables, so the real work was getting the same counting rules as the Work Orders list and checking each figure by hand. The Admin Approval fix was small in code but needed a change in design, so the same mistake cannot come back.

## How

**Work Order Progress report**: one query reads the live pieces' stage rows and groups them by product line and production stage. A piece counts while it is live (not rejected, product not denied). A stage step is done when COMPLETED or SKIPPED. The system stages HOLE and Dispatch are left out, as they are in the progress dialog and the Order Status report. Dates are days in Asia/Kolkata. The work order's overall percent is finished stage steps over all stage steps, the same figure the Work Orders list shows, and repeats on each row as "WO Progress".

**Remaining and waiting**: "Remaining Pieces" is Pieces minus Completed. "Pieces Waiting for This Stage" is worked out per piece from its own route: it is pending here and the stage before it is done or skipped (a first stage has no previous one). "Remaining Area" is the stage's total area minus the area done. "Stage Completion % of Work Order" is the row's completed steps over every stage step of the work order, so the rows add up to the overall percent.

**Photos buttons**: the shared Photos button already greys itself out when a row says it has no photos. The two tables just never sent the flag. A photo counts if it belongs to the booking, or to the enquiry the booking came from: the same rule the Photos window and the Final Invoice list use.

**Read from photo**: each photo row is compared with the sizes saved on the booking (same overall width and height, and the same quantity, on the same product or, for an unknown glass, the same thickness). Only rows that match are skipped. The merge runs outside a state updater, because React may run an updater twice and fire the message twice.

**Admin Approval**: the modal builds one typed `ApprovalConfirmation`. A Party approval is the advance. An Admin approval is its payment details and delivery details, both required, with optional remarks. For Admin approval, the payment, promised date and slip type are checked first, then the slip is uploaded once, then `onConfirm` is called once.

**Removed UI**: the Lot ID field is commented out, not deleted, so it can be restored.

## Decisions made

**Work Order Progress is a standard report: one filter bar and one table.** An earlier version stacked charts, cards and five tables on one page. No other report looks like that, so it was rebuilt to match the existing reports. Rejected and replaced pieces are now columns. Invoices and dispatch are not in this report; the Order Status Register shows them.

**One row per product and stage.** This gives stage-wise and product-wise progress in one table. The overall percent is repeated on each row, as Order Status repeats PI amounts.

**System stages are left out.** The Work Orders list does not filter them, but your data has none (0 of 404 stage rows), so the percentages match. A work order with such a row would show a slightly different percent in the list.

**The progress bar is only in the Stage Progress column on screen.** The downloads keep a plain percent. Mobile shows text.

**The shares can differ slightly from the whole-number WO Progress.** Each share is rounded to one decimal, so the shares can add up to 44.7% while WO Progress shows 44%.

**Photo rows are matched by sizes, not by glass name.** If someone edits a saved size after saving it, the photo row no longer matches and is offered again. This leans towards showing a row the user can delete, never towards hiding one.

**Color is required, and "Colorless" counts as a choice.** Some products are priced only without a colour, so they could not be booked otherwise. Older products saved with no colour still open as Colorless.

**Color is required on the screen only.** The server still accepts an item with no colour, because replicate and history import do not use this modal.

**Lot number stays in the Products table.** It is per product. The details field joined every product's lots into one line.

**Transaction Date is required for bank transfer and cheque.** This matches UPI and cash. An old client that does not send it will get an error. Older records have no date, so the read-only view leaves that line out for them.

**"Last Payment Date" was not renamed.** It means the date of the last payment, not the field.

**A failed slip upload now stops the approval.** Before, it showed an error and approved anyway.

## Bugs found while testing (already there before, not fixed here)

**1. A dormant Order Booking endpoint uses old column names.** The insert and update of a booking item in `order-booking.repository.ts` (`order_booking_id`, `product_id`) point at columns that no longer exist. The frontend never calls that endpoint, so nobody hits it today. It would fail if called directly.

**2. Existing type and lint errors.** Lint reports `any` types and a raw `toFixed` in `approve-party-modal.tsx`, and a React hook error in `report-table.tsx`. They are not in the lines this PR changed.

## Notes

- All changes were tested in the browser.
- Run `npm install` in the frontend if the build says `leaflet` is missing. It was already in `package.json`; it was missing only from `node_modules`.
- The photo merge fix has no unit test in the repo, because the frontend has no test runner.

## Next steps

- Add Completed Qty and Completed SQM columns to the Production Plan Daily Production report once their meaning is decided.
- Decide whether to tune the database pool (a shorter idle timeout, and the keep-alive on).

## Test plan

**Automated tests:**
- [x] Frontend type check (`npx tsc --noEmit -p tsconfig.app.json`) and `npm run build`: no errors.
- [x] Photo flags on a scratch Postgres (9 checks): a booking's own photo, an enquiry's photo, no photo, and a photo on the wrong screen or another company, for both lists.
- [x] Transaction date rules (10 checks): refused without a date, accepted with one, bad format refused, UPI and cash unchanged, and the date is saved and returned for a PI bank transfer and a Final Invoice cheque.
- [x] Photo merge logic (11 checks): new sizes of the same glass are added, a re-read adds nothing, mixed photos add only new rows, different quantity or thickness counts as new.
- [x] Typed approval: five wrong confirmations (no delivery, no payment, no date, no type, no advance) fail to compile.

**Manual checks (local):**
- [x] Reports → Production Plan → Work Order Progress: pick a work order, table, progress bar, the new columns, Excel and PDF. Tested in the browser.
- [x] Order Booking and Work Orders tables: Photos button enabled only when there are photos. Tested in the browser.
- [x] Order Booking: read from photo on a booking that already has the same glass adds the new sizes. Tested in the browser.
- [x] Add Product: Color required, "Colorless" accepted, error shown. Tested in the browser.
- [x] Work Order view: Lot number shown once, in the Products table. Tested in the browser.
- [x] PI Admin Approval and Final Invoice payment: Transaction Date for bank transfer and cheque. Tested in the browser.
- [x] Purchase Payment modal and Purchase view: "Transaction Date" label. Tested in the browser.
- [x] PI Admin Approval with a payment slip attached: approves, with the promised delivery date sent. Tested in the browser.
- [ ] A work order that has a HOLE or Dispatch stage row - your data has none, so it is covered only by the scratch database check.
