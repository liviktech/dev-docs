# 2026-11-24 - purchase-preview-order-booking-approval-and-ui-enhancements (frontend)

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
| `src/features/inventory/components/purchase-sheet.tsx` | Redesigned Purchase Order preview sheet: typography and container styles aligned with Order Booking and PI Status sheets; dynamic company profile via `useCompanyProfile`; dynamic supplier billing info; removed header Works section; formatted CIN as `CIN NO : ... , MSME : ...`; renamed to `Supplier (Bill From) :` and `Supplier Invoice :`; moved contact, GSTIN, and State into Consignee section; detailed product specifications table; subtotal, additional charges, CGST/SGST/IGST breakdown, and round off; removed static Terms & Conditions from preview, print styles, and PDF generation. |
| `src/features/inventory/purchase-types.ts` | Added supplier address, GSTIN, state code, phone, and email fields to inventory purchase types. |
| `src/features/inventory/purchase-cost.ts` | Added helper calculations for additional charges and tax splits in purchase sheets. |
| `src/features/order-booking/booking-header-form.tsx` | Moved "From Enquiry / Direct" radio options directly adjacent to the "Order Booking Type" heading. |
| `src/features/order-booking/header-detail-tab.tsx` | Added "Manager Approval" column (`Approved` / `Pending`); added "Approve" kebab action with confirmation dialog; added dedicated "View" option for confirmed orders. |
| `src/features/order-booking/capabilities.ts` | Added `approveBooking` permission check; restricted "Create PI" to manager-approved confirmed orders; restricted "Clone" to confirmed orders only. |
| `src/features/order-booking/api.ts` | Added `managerApprovedAt` and `managerApprovedByName` to booking types; added `approveOrderBooking` API call (`PATCH /order-bookings/:id/approve`). |
| `src/features/auth/permissions.ts` | Added `ORDER_BOOKING_APPROVE` permission identifier. |
| `src/features/pi-status/new-pi/new-pi-page.tsx` | Added gate preventing PI creation when navigating to an unapproved confirmed booking with a clear notification. |
| `src/features/pwa/order-booking/mobile-order-detail.tsx` | Mobile view: enforces manager approval before enabling "Create PI"; disables "Print Work Order" for draft work orders. |
| `src/features/order-booking/work-order-print-menu.tsx` | Disabled "Print Work Order" for draft work orders with a helpful tooltip. |
| `src/features/order-booking/work-order-row-menu.tsx` | Disabled "Print Work Order" in Work Orders table row menu for draft work orders. |
| `src/features/pwa/work-orders/mobile-work-order-detail.tsx` | Disabled "Print Work Order" in mobile work order detail for drafts. |
| `src/features/ai-extraction/photo-review-panel.tsx` | Read Order from Photo: stopped automatic extraction on modal open; multi-photo enquiry picker; large preview pane with Rotate Left and Rotate Right buttons; canvas-based rotated image upload; restricted "Take a Photo" to mobile touch devices (`(hover: none) and (pointer: coarse)`). |
| `src/features/enquiries/enquiries-page.tsx` | Omitted `onEdit` handler for Approved and Converted enquiries so read-only records do not show misleading edit prompts. |
| `src/features/enquiries/enquiry-row-menu.tsx` | Removed misleading "Edit" button for view-only enquiries; hides the kebab menu completely when no row actions are available. |
| `src/features/notifications/record-notifications-dialog.tsx` | Updated PI Status notification dialog to use compact, dense row cards for Section 1 ("PI created") and Section 2 ("Approval reminders"). |
| `tests/rbac-capabilities.test.ts` | Added unit tests covering Approve, Clone, and Create PI permission rules. |

## Why

1. **Purchase Preview Completeness & ERP Handoff**: The purchase preview is printed and handed to suppliers who enter it directly into their ERP systems. It requires complete, professional business details: dynamic company credentials, supplier billing and dispatch details, item descriptions with HSN/SAC codes, exact rates, quantities (including sq.ft where applicable), itemized additional charges (freight, loading, insurance, unloading), and GST breakdowns. Standard customer sales Terms & Conditions were inappropriate for supplier purchase orders and added unnecessary clutter.
2. **Manager Approval Governance**: Business governance requires a manager to sign off on confirmed order bookings before sales can generate a legal Proforma Invoice. This mirrors the existing approval workflow on proformas.
3. **Usability of AI Photo Reading**: Uploaded or captured enquiry slips are frequently oriented sideways or upside down. Previously, the AI extraction executed automatically on load, causing extraction errors on unrotated images. Users needed manual preview, rotation, and re-reading controls.
4. **Clarity in Row Actions**: Presenting an "Edit" option for enquiries that are already converted or approved misled sales representatives into thinking they could modify immutable records.
5. **Roles & Access Visibility**: The missing `$` parameter prefix in the SQL pagination query caused newly created roles on the first page to be skipped by the database driver, giving users the impression that role creation failed.

---

## How

- **Purchase Preview layout & dynamic data**:
  - Dynamically fetches the logged-in company details via `useCompanyProfile`, rendering legal name, email, website, `CIN NO : ... , MSME : ...`, phone, GSTIN, and state code.
  - The backend query (`getInventoryPurchaseById`) joins `core.party` to return dynamic supplier billing details (address, GSTIN, state, phone).
  - Cleaned up top header: removed redundant Works section; formatted CIN and MSME in a single line separated by commas.
  - Formatted supplier header as `Supplier (Bill From) :` and invoice labels as `Supplier Invoice :` and `Invoice Date :`.
  - Moved consignee contact, GSTIN, and State into the Consignee section alongside storage location and remarks.
  - Itemized table renders serial number, item code, description, HSN/SAC, quantity/sq.ft, unit rate, and total line amount.
  - Summary block computes subtotal, itemized additional charges, CGST/SGST/IGST breakdown, and round off.
  - Completely removed the Terms & Conditions section from UI preview, print stylesheets, and PDF generation.

- **Manager Approval Flow**:
  - Confirmed bookings default to `Pending` approval.
  - Users with `order_booking_approve` can click "Approve" in the kebab menu, which opens a confirmation dialog and dispatches `PATCH /order-bookings/:id/approve`.
  - The server records `manager_approved_at = NOW()` and `manager_approved_by = req.user.id`.
  - PI picker endpoints and frontend navigation guards enforce that only approved confirmed orders can progress to PI generation.

- **Read Order from Photo Flow**:
  - Stopped auto-reading on open. If an enquiry has multiple photos, the user selects which photo to review first.
  - Displays a full preview with Rotate Left (-90deg) and Rotate Right (+90deg) buttons.
  - When rotating and clicking "Read Photo", an offscreen HTML canvas renders the rotated image, exports a JPEG blob, uploads it as a new attachment, and feeds that rotated copy to the AI extractor.
  - "Take a Photo" is restricted to touch devices using CSS media query evaluation `(hover: none) and (pointer: coarse)`.

- **Enquiry Row Actions Refinement**:
  - In `enquiries-page.tsx`, `onEdit` is omitted (`undefined`) when `isViewStatus || viewOnlyAccess || !caps.canEdit`.
  - In `enquiry-row-menu.tsx`, if no actions exist on a row, the kebab button returns `null` instead of rendering an empty menu.
  - In `header-detail-tab.tsx`, confirmed orders receive an explicit "View" action (with Eye icon) for opening the booking details in view mode.

- **PI Notifications Modal**:
  - Refactored `record-notifications-dialog.tsx` to display compact row cards with concise date and badge alignments.

---

## Decisions made

2. **Removed Terms & Conditions from Purchase Preview**: Purchase orders are given to suppliers and governed by bilateral supply agreements; the customer sales T&C was irrelevant and took up nearly half a printed page.

3. **Rotating photo creates a new photo record**: The storage system treats uploaded files as immutable. Rotating an image writes a new rotated copy rather than attempting an in-place mutation.

## Bugs found while testing (already there before, not fixed here)

1. **Enquiry Edit button on locked records**: Prior to this PR, clicking "Edit" on an Approved or Converted enquiry opened the modal in read-only mode without explaining why editing was unavailable. Fixed in this PR by hiding the action and removing empty kebab menus.

2. **Role table listing failure on creation**: Roles created in the UI were successfully inserted into the database but omitted from the list response due to the pagination offset bug. Fixed in this PR.

---

## Notes

- By default, existing custom roles do not have `order_booking_approve`. Super Admins and Admins have access, while other roles can be granted this permission in Settings > Roles.
- Both frontend and backend builds compile cleanly with zero TypeScript errors.

---

## Next steps

- Configure role assignments for `order_booking_approve` for plant managers and sales heads in production.
- Consider adding the "Manager Approval" column to the PWA mobile order booking list in a future update.

---

## Test plan

**Automated tests:**
- [x] Frontend TypeScript compilation & build: `npx tsc -b && npx vite build` passed with 0 errors.
- [x] Backend route rights test: `tests/order-booking-route-rights.test.ts` passed (verifying `PATCH /order-bookings/:id/approve` requires `ORDER_BOOKING_APPROVE`).

**Manual checks (local):**
- [x] **Purchase Preview**:
- [x] **Order Booking & Manager Approval**:
- [x] **Read Order from Photo**:
- [x] **Enquiry Kebab Menu**:
- [x] **Roles & Access**:
- [x] **PI Status Notifications**:
