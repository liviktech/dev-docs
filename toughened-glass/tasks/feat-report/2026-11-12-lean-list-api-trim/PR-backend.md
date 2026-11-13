# 2026-11-12 - lean-list-api-trim (backend)

## Summary

Several list endpoints returned fields that no screen reads, or ran extra subqueries just to build those fields. This PR trims them so each list returns only what its table and cards show. Full objects still come from the by-id reads that the view, edit and print dialogs already use. The response shape for callers that do not opt in is unchanged. It also fixes the Inventory Purchase list so it returns `paymentStatus` (the frontend needed it and the lean list had dropped it), and it resolves a merge conflict in the purchase service.

**Dispatch `job_type` and Order Booking `projectName` are not removed on purpose** - see "Decisions made" below. This was a choice, not a miss.

Type check (`npx tsc --noEmit`) passes. I did not run the queries against the database or open the app, so the trimmed SELECTs are checked by types and by reading the frontend code only.

## What changed

| File | Change |
|---|---|
| `src/modules/order-booking/order-booking.repository.ts` | Removed the `EXISTS(... order_photo ...) AS is_photo` subquery from the `view=table` list. |
| `src/modules/order-booking/order-booking.service.ts` | Removed `isPhoto` from the paged and non-paged `view=table` mappers. The full list mapper still returns it. |
| `src/modules/order-booking/order-booking.types.ts` | Removed `isPhoto` from the table row type. |
| `src/modules/work-order/work-order.repository.ts` | Removed the `is_photo` subquery and the `order_booking_id` column. `orderBookingNo` now comes from `COALESCE(ob.order_code, wo.order_code)`. |
| `src/modules/work-order/work-order.types.ts` | `WorkOrderListItem` has `orderBookingNo` only. `orderBookingId` and `isPhoto` are gone. |
| `src/modules/dispatch/dispatch.repository.ts` | Removed `d.remarks` from the table select and `request_requested_at` from the delivery columns. |
| `src/modules/dispatch/dispatch.types.ts` | Removed `remarks` and `request_requested_at` from the row types. |
| `src/modules/production-plan/production-plan.repository.ts` | Removed the 4-join `STRING_AGG ... AS search_text` subquery from `listPlans`. |
| `src/modules/production-plan/production-plan.service.ts`, `production-plan.types.ts` | Removed `searchText` and `search_text`. |
| `src/modules/master/stock-option/stock-option.config.ts`, `stock-option.repository.ts` | New optional `listFieldKeys` in the config. The paged list only selects those extra fields. Suppliers set it to `phone` and `email`. |
| `src/modules/master/glass-type/*` | List selects `id, code, name, is_active` only (new `GlassTypeListRow` and `GlassTypeListItem`). |
| `src/modules/master/glass-thickness/*` | List selects `id, value_mm, is_active` only (new `GlassThicknessListRow`). |
| `src/modules/master/item/*` | List selects only the columns the Items table shows (new `ItemListRow` and `ItemListItem`). |
| `src/modules/master/role/*` | List uses an explicit column select (new `RoleListRow`). |
| `src/modules/master/payment-term/mode/*` | List uses an explicit column select (new `PaymentModeListRow`). |
| `src/modules/superadmin/superadmin.repository.ts`, `.types.ts`, `.service.ts` | Company list no longer selects `created_at` and `updated_at` (new `CompanyListRow`). |
| `src/modules/e-way-bill/e-way-bill.repository.ts`, `.types.ts`, `.service.ts` | The batch lookup `listActiveEwayBills` was `SELECT *` (20 columns). It now selects `id, final_invoice_id, ewb_number` (new `ActiveEwayBillRow`). The single-invoice read and the logs are unchanged. |
| `src/modules/inventory-purchase/inventory-purchase.repository.ts` | The list select now includes `t.paid_amount`, so the status can be worked out on the server. |
| `src/modules/inventory-purchase/inventory-purchase.types.ts` | `paymentStatus` is on the lean list item. `paid_amount` is on the list row. Removed a duplicate `PurchasePaymentStatus` type and a duplicate `paymentStatus` field on the summary. |
| `src/modules/inventory-purchase/inventory-purchase.service.ts` | `toPurchaseListItem` sets `paymentStatus` using the existing `paymentStatusOf(totalCost, paidAmount, paymentRecorded)`. Merge conflict resolved (see "How"). |

## Why

The audit started from HAR captures of Purchase, Sales Visits, PI Status, Final Invoice, Order Booking, Work Orders, Dispatch, Production Plan and the master lists. Each list was compared with what its table, cards and dialogs read. Anything computed or returned but never read was dropped, which also removes per-row subqueries (photo checks, the search text aggregate) that cost database time on every page load. The easy part was dropping plain columns. The real work was checking every reader first, including mapper-derived flags and mobile screens, before removing a field.

## How

**Lean rows with explicit types**: Each trimmed list has its own row and item type (`Pick`, `Omit` or a new `...ListRow`), so the compiler flags any reader that still expects a removed field.

**Opt-in behaviour kept**: Order Booking keeps `isPhoto` in its full list. Only `view=table` is lean. Callers that do not ask for the lean view get the same shape as before.

**Server-side search stays**: Production Plan search still works. It filtered on `search_text` inside the query and never needed to send it to the client.

**Suppliers extras**: `listFieldKeys` limits which configured extra fields the paged list selects. Other stock-option masters without it behave as before.

**Purchase merge conflict**: The conflict in `inventory-purchase.service.ts` was between my 2 argument `paymentStatusOf(paid, total)` and the incoming 3 argument `paymentStatusOf(totalCost, paidAmount, paymentRecorded)`. I kept the incoming one, deleted mine, and removed the duplicate imports, type and `paymentStatus` fields the merge left behind. The lean list does not return `paidAmount` or `remainingAmount`; the summary and detail still do.

## Decisions made

**Dispatch `job_type` is kept in the list.** The Dispatch details modal reads `record.jobType`. Removing it would need the modal to load by id, which is a behaviour change outside this PR.

**Order Booking `projectName` is still returned.** Nothing in the table reads it, but I did not drop it without a go ahead. Open question for the team.

**Purchase `paymentStatus` tolerance.** My first version treated the purchase as paid when paid was within 0.005 of the total. After the merge the incoming rule applies, which needs paid to be at least the total. A tiny rounding gap could now show as partly paid.

**Non-paged dropdown lists and by-id reads were left alone.** They feed edit dialogs and dropdowns, and the saving is small.

## Bugs found while testing (already there before, not fixed here)

**1. Purchase edit request endpoints are missing.** The frontend calls `GET /inventory/purchases/edit-requests` (and raise, approve, reject), but these routes do not exist in the backend, so the call returns 400. It was like this before. The frontend now stops retrying the call (see the frontend PR), but the endpoints are still not built.

## Notes

- Not audited: Parties, Users, Sales Reps, Invoice Terms, Bank Accounts, Reports, Employee and Attendance, FIR, Notifications.
- Production Plan, Order Booking and Work Orders share dispatch and PI code paths, so a quick look at those screens after deploy is worthwhile.
- `dist/` is build output and was not edited.

## Next steps

- Build the purchase edit request endpoints, or hide the edit request badge until they exist.
- Decide on dropping `projectName` (Order Booking) and `job_type` (Dispatch list).
- Audit the modules listed under Notes.

## Test plan

**Automated tests (no test files added):**
- [x] `npx tsc --noEmit` in the backend: passes with no errors after the merge conflict fix.
- [ ] No unit or integration tests cover these list queries, so none were run.

**Manual checks (local):**
- [ ] Open each trimmed list (Order Booking, Work Orders, Dispatch, Production Plan, the seven master lists, Purchase, e-Way Bill lookup) and confirm the rows load and look the same. Not done, I did not run the app or the queries.
- [ ] Production Plan search by product or customer still finds rows. Not done, needs a run against the database.
- [ ] Purchase list shows the right Unpaid, Partially Paid and Paid tags. Not done.
