# 2026-11-23 - created-by-approved-by (frontend)

## Summary

List tables had no Created By or Approved By columns, or showed Approved By in different places. This PR adds one shared column helper and uses it on about 30 list screens, so the two columns always sit just before Actions and look the same. It also passes the new `createdByName` and `approvedByName` fields through the screens that rebuild their rows by hand.

I checked that `npm run build` passes (Vite and `tsc -b`). I did not open the screens in a browser.

**The Actions column stays last on purpose** - see "Decisions made" below. This was a choice, not a miss.

## What changed

| File | Change |
|---|---|
| `src/components/ui/audit-columns.tsx` (new) | `auditColumns<T>({ created, approved })` returns the Created By and Approved By column definitions. Left aligned, no sorting, shows a dash when empty. |
| `src/features/enquiries/enquiries-page.tsx`, `enquiries/api.ts` | Both columns. `Enquiry` carries the two names. |
| `src/features/order-booking/header-detail-tab.tsx`, `order-booking/api.ts` | Created By only on the Order Booking table. The Work Order list item type carries both names. |
| `src/routes/work-orders.tsx` | Both columns on the Work Orders table. |
| `src/features/pi-status/components/pi-status-table.tsx`, `pi-status/api.ts`, `pi-status/types.ts` | Both columns on PI Status. The row mapper in `api.ts` now copies the two names (it rebuilds each row by hand and was dropping them). |
| `src/features/final-invoices/*` | Created By added. The existing Approved By column moved to the end through the helper. The row mapper copies `createdByName`. |
| `src/features/dispatch/components/dispatch-table.tsx`, `dispatch/types.ts` | Approved By moved to the end through the helper. Created By is not shown, as asked. |
| `src/features/inventory/purchase-tab.tsx`, `purchase-return-tab.tsx`, `purchase-return-types.ts` | Both columns. The return summary type gained `approvedByName`. |
| `src/features/procurement/procurement-tabs.tsx` | Indent and Purchase Request tables: Created Date first, then Created By and Approved By. |
| `src/features/production-plan/production-plan-panel.tsx`, `production-plan/api.ts` | "Planned By" is replaced by Created By and Approved By at the end. |
| `src/features/fir/fir-page.tsx`, `fir/api.ts` | Created By on FIR. |
| `src/features/attendance/attendance-page.tsx`, `attendance/types.ts` | Created By on Employees. |
| `src/features/inventory/customer-material-tab.tsx`, `locations-tab.tsx`, `customer-material.ts`, `inventory/types.ts` | Created By on Customer Material and Storage Locations. |
| `src/features/stock-masters/stock-master-page.tsx`, `stock-masters/types.ts` | Created By on the 14 shared masters. |
| `src/routes/party.tsx`, `features/parties/api.ts`, `parties/types.ts` | Created By on Party (the mapper copies the name). |
| `src/features/products/product-details-tab.tsx`, `products/api.ts`, `products/types.ts` | Created By on Product Details (the mapper copies the name). |
| `src/features/items/items-page.tsx`, `items/types.ts` | Created By on Items. |
| `src/routes/admin.work-centers.tsx`, `admin.shifts.tsx`, `admin.tax-profiles.tsx`, `admin.charges.tsx`, `admin.order-charges.tsx`, `admin.purchase-charges.tsx`, `admin.payment-terms.tsx`, `admin.invoice-terms.tsx`, `admin.sales-reps.tsx` and their `types.ts` / `api.ts` | Created By on each master page. `sales-reps/api.ts` copies the name. |
| `src/features/roles/roles-access-page.tsx`, `roles/types.ts` | Created By on the Roles table only. |
| `src/routes/admin.invoice-terms.tsx`, `src/routes/admin.payment-terms.tsx`, `src/features/order-booking/header-detail-tab.tsx` | Removed the unused `Eye` import so `tsc -b` passes (see "Bugs found while testing"). |
| `src/components/ui/select.tsx` | no real change from this work. The sticky "Add New" row in this file is a separate change that landed in the same commit. |
| `src/features/production-scan/components/cutting-entry-dialog.tsx`, `src/features/pwa/work-orders/mobile-piece-detail.tsx` | no real change from this work. The offcut boxes locking when there is no offcut is a teammate's change in the same commit. |
| `src/features/sales-visits/sales-visits-page.tsx` | no real change from this work. A removed "View" row action in the same commit is not part of it. |

## Why

The owner wanted every list to show who created and who approved each record as the last columns. Writing the two columns by hand on 30 screens would have drifted (different widths, empty text, sorting), so they come from one helper.

## How

**One helper**: `auditColumns<T>()` takes `{ created, approved }` (both on by default) and returns plain column definitions. A screen adds `...auditColumns<Row>({ approved: false })` just before its Actions column. Header and cell use the existing `meta: { align: 'left' }` rule from the table alignment skill.

**Field names**: the helper reads `createdByName` and `approvedByName` from the row. Screens that pass the API response straight through needed only a type change. Screens that rebuild each row by hand (PI Status, Final Invoices, Party, Product, Sales Rep) needed the mapper to copy the new fields too.

**Kept the same on purpose**: row actions, sorting of the existing columns, and the API calls. Only columns, types and mappers changed.

## Decisions made

**Actions stays the last column.** The request said the two columns should be last. The table alignment rules say Actions is last and right aligned, so the two columns sit directly before it.

**Created By only where there is no approval.** Order Booking, FIR, Employees, Customer Material, Locations, Party, Product, Items, the masters and Roles use `approved: false`.

**Approved By shows its own source on Purchase Request.** That table has no approver column, so Approved By shows the existing "accepted by" name.

**Dispatch and Supplier Payments follow the owner's list.** Dispatch shows Approved By only. Supplier Payments shows neither column.

## Bugs found while testing (already there before, not fixed here)

**1. Unused `Eye` imports broke `npm run build`.** `header-detail-tab.tsx`, `admin.invoice-terms.tsx` and `admin.payment-terms.tsx` imported `Eye` but no longer used it, because the View row action had been removed in the same commit. Vite built fine, but `tsc -b` failed. I removed the three imports. The removed View actions were not part of this work.

**2. Payment Terms shares one table for three tabs.** Conditions, Modes and Restrictions use the same column set, but only Modes stores a creator, so Created By is a dash on the other two.

## Notes

- Existing records have no creator stored, so they show a dash. Only records created after the backend change carry a name.
- The build prints a warning that the main bundle is about 4.8 MB. It was there before.

## Next steps

- Sales Visits Created By (needs the backend change first).
- Decide what old blank rows should show.

## Test plan

**Automated tests (build passes, no UI tests cover this):**
- [x] `npx tsc --noEmit -p tsconfig.app.json`: passes with no errors.
- [x] `npm run build` (Vite and `tsc -b`): passes after the `Eye` import fix. It failed before with three `TS6133` errors.

**Manual checks (local):**
- [ ] Open each screen and read the Created By and Approved By columns. Not done, because I did not open the screens in a browser.
- [ ] Create one new record on a master page and check the name appears. Not done for the same reason.
- [ ] Check the columns on the mobile (PWA) lists. Not covered by this change.
