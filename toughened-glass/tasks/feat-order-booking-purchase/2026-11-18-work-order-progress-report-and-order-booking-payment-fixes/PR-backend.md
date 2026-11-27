# 2026-11-18 - work-order-progress-report-and-order-booking-payment-fixes (backend)

## Summary

This PR is the second batch of work from one day (the first batch is in `docs/PR-department-daily-report-po-purchase-columns-and-fixes.md`). The main piece is a new **Work Order Progress** report. The rest is a set of fixes in Order Booking, the PI Admin Approval and the payment screens.

Before this PR there was no report showing how far one work order has progressed. The Photos button in the Work Orders and Order Booking tables was enabled on every row, even for orders with no photos. Reading a photo into the Add Product table skipped any glass the booking already had, so new sizes of the same glass were dropped. Color was optional in Add Product. The Work Order view showed the Lot number twice. Bank transfer and cheque payments had no transaction date. Admin Approval with a payment slip attached failed, because the promised delivery date was left out of the request.

Each change keeps the existing flows working. All changes were tested in the browser. Frontend and backend type checks pass. The new report's queries, the photo flags, the transaction date rules and the photo merge logic were also run on a scratch local Postgres or as plain functions.

**Moving the slip upload into the approval request on the server is not included on purpose** - see "Decisions made" below. This was a choice, not a miss.

## What changed

| File | Change |
|---|---|
| `src/modules/report/work-order-progress/work-order-progress-report.schema.ts` (new) | Validates the `workOrderId` query. |
| `src/modules/report/work-order-progress/work-order-progress-report.repository.ts` (new) | The queries for one work order: header, one row per product and stage, rejected and replacement counts, downtime. |
| `src/modules/report/work-order-progress/work-order-progress-report.service.ts` (new) | Builds the rows and every figure (remaining, waiting, area, percentages, status). |
| `src/modules/report/work-order-progress/work-order-progress-report.controller.ts` (new) | Thin HTTP layer. |
| `src/modules/report/report.routes.ts` | New `/reports/work-order-progress` route with the same Reports view permission as the other reports. |
| `src/modules/order-booking/order-booking.repository.ts`, `order-booking.service.ts`, `order-booking.types.ts` | The Order Booking table now returns whether the booking has photos (its own or its enquiry's). |
| `src/modules/work-order/work-order.repository.ts`, `work-order.types.ts` | The Work Orders list returns the same photo flag. |
| `src/modules/payment-detail/payment-detail.schema.ts`, `payment-detail.service.ts` | A bank transfer or cheque now requires a transaction date, and it is saved. Before, only UPI and cash stored one. |

## Why

The Work Order Progress report is a new reporting need: one place to see how far a work order is, stage by stage and product by product. The Order Booking, payment and approval changes came from review feedback and from bugs found while testing: the Photos button that never greyed out, photos with new sizes being thrown away, the double Lot number, and the Admin Approval failure.

Most of the work was in the report. The data already existed in the production tables, so the real work was getting the same counting rules as the Work Orders list and checking each figure by hand. The Admin Approval fix was small in code but needed a change in design, so the same mistake cannot come back.

## How

**Work Order Progress report**: one query reads the live pieces' stage rows and groups them by product line and production stage. A piece counts while it is live (not rejected, product not denied). A stage step is done when COMPLETED or SKIPPED. The system stages HOLE and Dispatch are left out, as they are in the progress dialog and the Order Status report. Dates are days in Asia/Kolkata. The work order's overall percent is finished stage steps over all stage steps, the same figure the Work Orders list shows, and repeats on each row as "WO Progress".

**Remaining and waiting**: "Remaining Pieces" is Pieces minus Completed. "Pieces Waiting for This Stage" is worked out per piece from its own route: it is pending here and the stage before it is done or skipped (a first stage has no previous one). "Remaining Area" is the stage's total area minus the area done. "Stage Completion % of Work Order" is the row's completed steps over every stage step of the work order, so the rows add up to the overall percent.

**Transaction Date**: the shared payment form shows it beside Payment Mode in the wide layout and under it in the narrow one. The server schema requires it for bank transfer and cheque, and the service stores it.

## Decisions made

**System stages are left out.** The Work Orders list does not filter them, but your data has none (0 of 404 stage rows), so the percentages match. A work order with such a row would show a slightly different percent in the list.

**The shares can differ slightly from the whole-number WO Progress.** Each share is rounded to one decimal, so the shares can add up to 44.7% while WO Progress shows 44%.

**Photo rows are matched by sizes, not by glass name.** If someone edits a saved size after saving it, the photo row no longer matches and is offered again. This leans towards showing a row the user can delete, never towards hiding one.

**"Last Payment Date" was not renamed.** It means the date of the last payment, not the field.

**A failed slip upload now stops the approval.** Before, it showed an error and approved anyway.

**Moving the slip upload into the approval request on the server is not included.** It is the cleanest design, but it is a bigger change, and a file store cannot be rolled back with the database anyway.

## Bugs found while testing (already there before, not fixed here)

**1. The project's seed script does not run on a fresh database.** `001_company.sql` does not set `company_code`, which a later migration made mandatory. This affects anyone setting up a new environment.

**2. A dormant Order Booking endpoint uses old column names.** The insert and update of a booking item in `order-booking.repository.ts` (`order_booking_id`, `product_id`) point at columns that no longer exist. The frontend never calls that endpoint, so nobody hits it today. It would fail if called directly.

**3. The database drops idle connections.** The backend logs "Unexpected PostgreSQL pool error: Connection terminated unexpectedly" when Neon closes an idle connection. It is usually harmless, but one request can fail if it uses a connection that has just been closed. The keep-alive query is off and idle connections are held for 5 minutes.

## Notes

- The report queries, the photo flags and the transaction date rules were run on a scratch local Postgres built from all migrations, with synthetic data. The work order fixture included a rejected piece with a replacement, a skipped stage, a HOLE system stage, open downtime and two invoices.
- Three read-only checks were made against the database in the backend `.env`: today's PI, the migrations applied, and that your real data has no system-stage rows. Nothing was written there.

## Next steps

- Decide whether to tune the database pool (a shorter idle timeout, and the keep-alive on).
- Decide whether Color should also be checked on the server.
- Fix the dormant Order Booking item endpoint, or remove it.
- Fix the seed script so a fresh database can be set up.
- Consider moving the payment slip upload into the approval request on the server later.

## Test plan

**Automated tests:**
- [x] Backend type check (`npx tsc --noEmit`): no errors.
- [x] Work Order Progress queries on a scratch Postgres: every row matched the hand-worked numbers, including the IST day edge, skipped and HOLE stages, waiting pieces, downtime, rejected and replacement counts, a second company getting "not found", and a draft work order with no pieces. The only two failing checks were my test's fault: an open downtime that keeps growing with the clock, and a rounding tolerance.
- [x] Transaction date rules (10 checks): refused without a date, accepted with one, bad format refused, UPI and cash unchanged, and the date is saved and returned for a PI bank transfer and a Final Invoice cheque.
- [x] Photo merge logic (11 checks): new sizes of the same glass are added, a re-read adds nothing, mixed photos add only new rows, different quantity or thickness counts as new.
- [x] Typed approval: five wrong confirmations (no delivery, no payment, no date, no type, no advance) fail to compile.

**Manual checks (local):**
- [x] PI Admin Approval and Final Invoice payment: Transaction Date for bank transfer and cheque. Tested in the browser.
- [ ] A work order that has a HOLE or Dispatch stage row - your data has none, so it is covered only by the scratch database check.
