# 2026-11-12 - lean-list-api-trim (frontend)

## Summary

The frontend types and a few screens still described list fields that no table reads, and some calls were wasteful (a polling progress list, a failing edit request call that kept retrying). This PR moves the types to match the lean backend lists, fixes a type error in Inventory Purchase, and stops the pointless calls. Screens look and behave the same.

Type check (`npx tsc --noEmit -p tsconfig.app.json`) passes. I did not open the app in a browser, so the screens are checked by types and by reading the code only.

## What changed

| File | Change |
|---|---|
| `src/features/order-booking/api.ts` | `WorkOrderListItem` has `orderBookingNo` and no `orderBookingId`. |
| `src/routes/work-orders.tsx` | Uses `row.orderBookingNo` where it used `orderBookingId` (about six places). |
| `src/features/production-plan/api.ts` | Removed `searchText` from `ProductionPlanListItem`. |
| `src/features/glass-thickness/api.ts` | `createdAt` and `updatedAt` are now optional. |
| `src/features/items/types.ts` | New `ItemListItem`, used in `PaginatedItemsResponse`. |
| `src/features/items/items-page.tsx`, `item-view-modal.tsx`, `add-to-product-dialog.tsx` | Use `ItemListItem`. In the view modal, `gstRate`, `isSales`, `isPurchase` and `description` now come from the by-id fetch. |
| `src/features/pwa/master/heavy-adapters.tsx` | `itemAdapter` is `MasterAdapter<ItemListItem, Item>`. |
| `src/features/superadmin/types.ts` | Removed `createdAt` from the company list type. |
| `src/features/e-way-bill/types.ts` | `ActiveEwayBill` is a Pick of `id`, `finalInvoiceId`, `ewbNumber`. |
| `src/features/e-way-bill/api.ts` | `fetchActiveEwayBills` returns `ActiveEwayBill[]`, mapped from the lean rows. |
| `src/features/inventory/purchase-types.ts` | Added `paymentStatus` to the `InventoryPurchaseListItem` Pick. |
| `src/features/inventory/purchase-tab.tsx` | `editingPaymentsOf` state is typed `InventoryPurchaseListItem \| null`. This fixes the type error. |
| `src/features/inventory/purchase-edit-requests.ts` | `purchaseEditRequestsQueryOptions` has `retry: false`, with a comment. |
| `src/features/production-scan/queries.ts` | The scan list is no longer polled every 20 seconds. It uses a 60 second stale time and refreshes on focus and after a scan, completion or rejection. |
| `src/features/dispatch/components/dispatch-table.tsx` | No real change. A Work Order Type column was tried and removed, so the file is back to the original. |

## Why

The backend lists were trimmed after checking what each table reads, so the frontend types had to follow or they would claim fields that never arrive. The Purchase type error appeared because the lean list had lost `paymentStatus`, which the payment dialog needs. The production scan list only feeds a Progress column, so downloading about 70 KB every 20 seconds with nobody clicking was waste.

## How

**Types follow the data**: Lean types (`ItemListItem`, `ActiveEwayBill`, `Pick` on the purchase and sales visit rows) make the compiler flag any code that reads a removed field.

**Detail fields from the by-id read**: The Items view modal already fetched the full item, so it now reads the detail-only fields from that result, not from the list row.

**Edit request calls**: `retry: false` stops the app from sending the failing `edit-requests?status=ALL` call twice per load or window. The call itself still happens once, because the backend endpoints do not exist yet.

## Decisions made

**Dispatch Work Order Type column was removed again.** It was added at first and then taken out on request, with no change to the API for it. `job_type` stays in the Dispatch list because the details modal reads it.

**Master `Products` was left as is on purpose.**

## Bugs found while testing (already there before, not fixed here)

**1. Purchase edit requests return 400.** `GET /inventory/purchases/edit-requests` is not built in the backend, so the badge never gets data. It was like this before. See the backend PR.

## Notes

- Not audited: Parties, Users, Sales Reps, Invoice Terms, Bank Accounts, Reports, Employee and Attendance, FIR, Notifications.

## Next steps

- Build the purchase edit request endpoints, or hide the badge until they exist.
- Confirm whether the PI Status production scan refresh timing is enough, or lengthen it.

## Test plan

**Automated tests (no test files added):**
- [x] `npx tsc --noEmit -p tsconfig.app.json` in the frontend: passes with no errors.
- [ ] No unit tests cover these screens, so none were run.

**Manual checks (local):**
- [ ] Items page: open a row in the view modal and confirm GST rate, sales and purchase flags and description show. Not done, I did not open the app.
- [ ] Work Orders page: confirm the order number shows in the table and in the modals. Not done.
- [ ] Inventory Purchase: open the payments dialog and confirm the status shows. Not done.
- [ ] Network tab on the Inventory page: confirm `edit-requests` is sent once, not twice. Not done.
