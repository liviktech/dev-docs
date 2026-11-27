# 2026-10-30 - dispatch-plan-loader-pi-gstin (backend)

## Summary

This PR holds all the uncommitted backend changes on `feat/report-goods`. There are five groups.

1. **Dispatch Plan can cover several Final Invoices.** Before, a plan was tied to one invoice (one plan per invoice, opened by invoice id). Now a plan is its own record with its own number (`DPL...`), and every line points at its own invoice and product. Two new migrations back this.
2. **Dispatch Plan number.** A new `DISPATCH_PLAN` entry in the Sequences master (default prefix `DPL`, start 101) numbers each plan.
3. **PI preview GSTIN.** The PI query now sends `gstin` on the billing and shipping address, so the PI preview can show the customer's GSTIN from master data.
4. **Vehicle and driver on the final invoice.** The final invoice response now carries the vehicle number and driver name from its dispatch, so the e-invoice sheet can print them.
5. **Search.** Enquiry search also matches visit code and enquiry source (code or name). Order Booking search also matches the PI code.

**UID/Aadhaar is not included on purpose** - see "Notes" below. This was a choice, not a miss.

## What changed

| File | Change |
|---|---|
| `src/database/migrations/108_dispatch_plan_no.sql` (new) | Adds `plan_no` to `invoice.dispatch_plan` and numbers the existing plans (oldest first) with the new `DISPATCH_PLAN` sequence |
| `src/database/migrations/109_dispatch_plan_multi_invoice.sql` (new) | Adds `final_invoice_id` to `invoice.dispatch_plan_line` (filled from the plan for old rows), a foreign key and an index on it, drops the one plan per invoice unique constraint, and makes `dispatch_plan.final_invoice_id` nullable |
| `src/api/route-mapper.ts` | Removed the two old mounts (`dispatchPlanListRouter` at `/dispatch-plans` and the per invoice router at `/final-invoices/:finalInvoiceId/dispatch-plan`). One `dispatchPlanRouter` is mounted at `/dispatch-plans` |
| `src/modules/dispatch-plan/dispatch-plan.routes.ts` | One router. `GET /` list, `GET /invoice-options`, `POST /` create, and `GET/PUT/DELETE /:planId`, `POST /:planId/submit`, `PATCH /:planId/approve`, `PATCH /:planId/reject`. Same permissions as before. The old `/eligible-invoices` route is gone |
| `src/modules/dispatch-plan/dispatch-plan.controller.ts` | Reads `planId` instead of `finalInvoiceId`. `listEligible` became `listInvoiceOptions` (takes `excludePlanId`). New `createPlan` (returns 201) |
| `src/modules/dispatch-plan/dispatch-plan.schema.ts` | `invoicePlanParamSchema` became `planParamSchema` (`planId`). New `planOptionsQuerySchema`. `planNo` added to the sortable fields |
| `src/modules/dispatch-plan/dispatch-plan.types.ts` | Rows and DTOs reshaped: plan has `plan_no`, `invoices` (list of invoice refs) and created by / created at. Lines carry their own invoice, product, thickness and invoiced quantity. `EligibleInvoice*` replaced by `PlanInvoiceOptionDto` with `plannedElsewhere` per product |
| `src/modules/dispatch-plan/dispatch-plan.repository.ts` | Header and items are looked up for many invoices at once. Each product also returns how much is already planned in other plans. The list query now returns one row per plan with its distinct invoices, and search and sort use the plan number and the first invoice. Lines are joined to invoice, work order, party and product. `ensurePlanInTx` replaced by `insertPlanInTx`, and `lockPlanInTx` locks by plan id. Line inserts include `final_invoice_id` |
| `src/modules/dispatch-plan/dispatch-plan.service.ts` | New `createPlan` (takes the next plan number and saves the lines in one transaction). `getPlan`, `savePlan`, `submitPlan`, approve, reject and delete work by plan id. `assertLines` checks each line's invoice is confirmed, the product belongs to the chosen invoice, and the quantity planned across all plans does not go over the invoiced quantity. `listInvoiceOptions` only returns invoices that still have something left to plan |
| `src/shared/utils/business-code.util.ts` | Added `DISPATCH_PLAN` (prefix `DPL`, start 101, `YYYYMMDD`) |
| `src/modules/master/sequence/sequence.schema.ts`, `sequence.types.ts`, `sequence.service.ts` | Added `DISPATCH_PLAN` to the entity list, the type, and the entity definitions ("Dispatch Plan", `DPL`, 101) |
| `src/modules/proforma-invoice/proforma-invoice.repository.ts` | Added `'gstin', ba.gstin` to `billing_address` and `'gstin', sa.gstin` to `shipping_address`. The `addresses` list already had `gstin` |
| `src/modules/final-invoice/final-invoice.repository.ts`, `final-invoice.types.ts` | Added `dispatch_vehicle_number` and `dispatch_driver_name` (read from `invoice.dispatch` for the invoice) |
| `src/modules/enquiry/enquiry.repository.ts` | The search box also matches `visit_code`, `enquiry_source`, and the source's name from `master.enquiry_source` |
| `src/modules/order-booking/order-booking.repository.ts` | The search box also matches the PI code of the booking's proforma invoice |

## Why

- Dispatch: one truck run often carries goods from more than one invoice, and a plan tied to one invoice could not show that. It works like a production plan, which already covers several work orders. A plan needs its own number so people can talk about it, and so the list can show it as the first column.
- PI GSTIN: the PI preview read GSTIN, state and state code from places the backend never filled, so GSTIN showed a dash. `master.address` already stores `gstin`, it was just not sent on the billing and shipping address.
- Vehicle and driver: the e-invoice sheet had these two boxes empty. The dispatch approval already records them.
- Search: users search by PI code, visit code or source and got no result.

## How

**Plans have their own id.** Before, the URL was `/final-invoices/:id/dispatch-plan` and the plan was created on first save. Now `POST /dispatch-plans` creates a plan and gives it a number, and all other calls use `/dispatch-plans/:planId`. The first save of a new plan generates the number inside the same transaction as the lines, so a failed save does not use up a number.

**Quantity rule.** For each product, the quantity in this plan plus the quantity in every other plan (excluding the plan being edited) cannot go over the invoiced quantity. `invoice-options` sends `plannedElsewhere` per product and hides invoices with nothing left, so the screen can show what is free.

**Old data.** Migration 109 copies the plan's old invoice onto each existing line, so old plans keep working. `dispatch_plan.final_invoice_id` is kept (nullable) and no longer drives anything. Migration 108 gives old plans a number, oldest first.

**GSTIN.** Same query, one extra key in each `jsonb_build_object`. No join and no new column. The frontend works out the state code from the GSTIN.

## Decisions made

**Old per invoice routes were removed, not kept next to the new ones.** The frontend in this same change uses only the new routes. Any other client still calling `/final-invoices/:id/dispatch-plan` or `/dispatch-plans/eligible-invoices` will break. Nothing else in the repo calls them as far as the search showed.

**`dispatch_plan.final_invoice_id` stays in the table.** It is nullable now instead of dropped, so the migration is safe on old rows and can be rolled back by hand if needed.

## Notes

- UID/Aadhaar has no column in `master.party` or `master.address`, so there is nothing to return. Adding it needs a new migration and a party form change, which was left out.
- The server needs a restart, and the two migrations need to run (`npm run migrate`), before the new dispatch plan routes work.
- With several invoices on one plan, sorting the plan list by invoice, work order or party uses only the first invoice (the smallest code). That is how the list was reshaped here, so it is a choice, not a bug.
- The dispatch plan rewrite, the sequence entry, the vehicle and driver fields and the two search changes were already in the working tree when this document was written. They are described from reading the diff and the two migration files.

## Next steps

- Add a `uid_aadhaar` field to the party in master data if the PI sheet should show it.
- Run migrations 108 and 109 on a copy of real data and check the old plans got a number and each line got its invoice.

## Test plan

**Automated tests (type check passing, no unit tests cover this):**
- [x] `npx tsc --noEmit`: no errors, run with all the changes above in the tree
- [ ] No test file covers the dispatch plan service, the PI query, or the search changes, so none was added

**Manual checks (local):**
- [ ] Run migrations 108 and 109 and check old plans have a `plan_no` and every old line has a `final_invoice_id` - **not done**, the migrations were not run
- [ ] Create a plan with lines from two invoices, submit, approve and reject it - **not done**, the API was not called
- [ ] Try to plan more than the invoiced quantity across two plans and check `PLAN_QUANTITY_EXCEEDED` - **not done**
- [ ] Call the PI detail and list endpoints and check `billing_address.gstin` and `shipping_address.gstin` are present - **not done**, the SQL was not run against a database
- [ ] Check the enquiry search finds a visit code and a source name, and the order booking search finds a PI code - **not done**
