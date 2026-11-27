# 2026-11-24 - purchase-preview-order-booking-approval-and-ui-enhancements (backend)

## Summary

Before this PR, several key operational flows across the ERP had UX gaps and missing business controls:
1. **Purchase Preview**: The purchase preview sheet was incomplete, relied on hardcoded company details, lacked dynamic supplier GSTIN, address, and state code details, missed item-level HSN/rate/sq.ft breakdowns and additional cost summaries, included a generic customer Terms & Conditions section not suited for suppliers, and deviated visually from Order Booking and PI Status preview layouts.
2. **Order Booking Governance**: Confirmed orders could be converted directly into Proforma Invoices (PI) without managerial review or approval.
3. **Read Order from Photo**: Opening the photo review modal triggered AI extraction immediately without letting users inspect the image or rotate sideways photos, and presented a "Take a Photo" option on desktop browsers where device cameras are unavailable.
4. **Order Booking Form Layout**: The "From Enquiry / Direct" selection was visually separated from the "Order Booking Type" header.
5. **Enquiry Row Actions**: Approved and Converted enquiries displayed an "Edit" action in the row menu, even though opening the modal opened it in read-only mode, causing confusion.
6. **Roles & Access Module**: Newly created roles did not display in the roles table due to an SQL query string interpolation bug in pagination offset parameters.
7. **PI Status Notifications**: Notification cards in the PI status notifications dialog were oversized and consumed excessive vertical space.

This PR addresses all of these areas across both the frontend and backend:
- Overhauls the **Purchase Preview** with dynamic company and supplier data, clean typography aligned with Order Booking and PI Status sheets, detailed financial/tax breakdowns, and removes irrelevant Terms & Conditions.
- Implements a complete **Manager Approval** workflow for confirmed orders with role-based permissions, table status indicators, kebab actions, and PI conversion gates.
- Redesigns the **Read Order from Photo** workflow with a manual read trigger, image rotation (Rotate Left / Right), canvas-based photo persistence, and mobile-only camera triggers.
- Fixes the **Enquiry kebab actions** to hide editing on locked enquiries and adds a dedicated "View" action for confirmed bookings.
- Fixes the **Role pagination SQL bug** so newly created roles appear immediately.
- Refines the **PI Status notification modal** with compact row cards.

---

## What changed

| File | Change |
|---|---|
| `src/database/migrations/181_order_booking_approve_rights_catalog.sql` (new) | Added `order_booking_approve` right into `core.rights_catalog` (initially inactive). |
| `src/database/migrations/182_order_booking_manager_approval.sql` (new) | Added `manager_approved_at` and `manager_approved_by` to `sales.order_booking`; backfilled existing bookings having a PI as approved; activated `order_booking_approve` in catalog. |
| `src/shared/rbac/permissions.ts` | Added `ORDER_BOOKING_APPROVE: "order_booking_approve"`. |
| `src/modules/order-booking/order-booking.routes.ts` | Added `PATCH /order-bookings/:id/approve` route guarded by `ORDER_BOOKING_APPROVE`. |
| `src/modules/order-booking/order-booking.controller.ts` | Added `approveBooking` controller handler. |
| `src/modules/order-booking/order-booking.service.ts` | Added `approveOrderBooking` service method: validates booking status is `CONFIRMED`, prevents re-approval, records approver ID and timestamp. |
| `src/modules/order-booking/order-booking.repository.ts` | Added `recordBookingManagerApproval`; updated booking list query to select approval columns; updated `view=pi-picker` query to filter only manager-approved confirmed bookings. |
| `src/modules/order-booking/order-booking.types.ts` | Added `managerApprovedAt` and `managerApprovedByName` to booking row and response types. |
| `src/modules/inventory/inventory-purchase/inventory-purchase.repository.ts` | Joined supplier party master details in `getInventoryPurchaseById` query to fetch supplier address, GSTIN, state code, phone, and contact person. |
| `src/modules/inventory/inventory-purchase/inventory-purchase.service.ts` | Mapped supplier address and tax details into inventory purchase response. |
| `src/modules/inventory/inventory-purchase/inventory-purchase.types.ts` | Added supplier address, GSTIN, and tax metadata to `InventoryPurchaseResponse`. |
| `src/modules/master/role/role.repository.ts` | Fixed SQL pagination bug: corrected `LIMIT ${pagedParams.length - 1} OFFSET ${pagedParams.length}` to `LIMIT $${pagedParams.length - 1} OFFSET $${pagedParams.length}`. |
| `tests/order-booking-route-rights.test.ts` | Added tests verifying route permission guards for `PATCH /order-bookings/:id/approve`. |
| `tests/rbac-permission-catalog.test.ts` | Added migration 181 to permission catalog validation test. |

---

## Why

1. **Purchase Preview Completeness & ERP Handoff**: The purchase preview is printed and handed to suppliers who enter it directly into their ERP systems. It requires complete, professional business details: dynamic company credentials, supplier billing and dispatch details, item descriptions with HSN/SAC codes, exact rates, quantities (including sq.ft where applicable), itemized additional charges (freight, loading, insurance, unloading), and GST breakdowns. Standard customer sales Terms & Conditions were inappropriate for supplier purchase orders and added unnecessary clutter.
2. **Manager Approval Governance**: Business governance requires a manager to sign off on confirmed order bookings before sales can generate a legal Proforma Invoice. This mirrors the existing approval workflow on proformas.
3. **Usability of AI Photo Reading**: Uploaded or captured enquiry slips are frequently oriented sideways or upside down. Previously, the AI extraction executed automatically on load, causing extraction errors on unrotated images. Users needed manual preview, rotation, and re-reading controls.
4. **Clarity in Row Actions**: Presenting an "Edit" option for enquiries that are already converted or approved misled sales representatives into thinking they could modify immutable records.
5. **Roles & Access Visibility**: The missing `$` parameter prefix in the SQL pagination query caused newly created roles on the first page to be skipped by the database driver, giving users the impression that role creation failed.

---

## How

- **Roles Pagination Bug Fix**:
  - In `role.repository.ts`, changed query string interpolation `LIMIT ${pagedParams.length - 1} OFFSET ${pagedParams.length}` to `LIMIT $${pagedParams.length - 1} OFFSET $${pagedParams.length}`.

## Decisions made

1. **Server does not hard-block direct PI creation**: Direct order booking creation automatically confirms and creates a PI in one flow. Hard-blocking unapproved bookings at the backend endpoint would break direct order bookings. The approval gate is enforced in the New PI picker (`view=pi-picker`), the Create PI shortcut, and the New PI route guard.

2. **Removed Terms & Conditions from Purchase Preview**: Purchase orders are given to suppliers and governed by bilateral supply agreements; the customer sales T&C was irrelevant and took up nearly half a printed page.

4. **Backfill migration only approves orders with an existing PI**: Existing confirmed orders that have not yet had a PI generated remain `Pending` so that managers can review them under the new governance rules.

---

## Notes

- Backend migrations `181_order_booking_approve_rights_catalog.sql` and `182_order_booking_manager_approval.sql` must be applied prior to deploying the backend.
- By default, existing custom roles do not have `order_booking_approve`. Super Admins and Admins have access, while other roles can be granted this permission in Settings > Roles.

## Next steps

- Configure role assignments for `order_booking_approve` for plant managers and sales heads in production.

## Test plan

**Automated tests:**
- [x] Backend TypeScript compilation: `npx tsc --noEmit` passed with 0 errors.
- [x] Backend route rights test: `tests/order-booking-route-rights.test.ts` passed (verifying `PATCH /order-bookings/:id/approve` requires `ORDER_BOOKING_APPROVE`).
- [x] Backend catalog test: `tests/rbac-permission-catalog.test.ts` passed (verifying migration 181 is present and mapped).

**Manual checks (local):**
- [x] **Purchase Preview**:
- [x] **Order Booking & Manager Approval**:
- [x] **Read Order from Photo**:
- [x] **Roles & Access**:
- [x] **PI Status Notifications**:
