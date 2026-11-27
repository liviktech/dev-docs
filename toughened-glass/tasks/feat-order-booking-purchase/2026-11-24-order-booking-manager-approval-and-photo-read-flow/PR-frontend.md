# 2026-11-24 - order-booking-manager-approval-and-photo-read-flow (frontend)

## Summary

Before this PR, any confirmed order could go straight to a Proforma Invoice (PI), nobody had to sign it off first. The "Read from photo" modal also read the enquiry photo by itself the moment it opened, with no way to turn a sideways photo first and no choice when an enquiry had several photos. A few screens also offered actions that did not make sense (Clone on drafts, Print Work Order on draft work orders, Take a photo on a desktop browser).

This PR adds a **Manager Approval** step for confirmed orders (backend and frontend), reworks the Read from photo flow, tidies four Order Booking / Work Order screens, and includes a change to the PI Status notification modal.

All changes were tested in the browser by the author. Automated results are listed in the Test plan.

**The server does not block PI creation for an unapproved order** - see "Decisions made" below. This was a choice, not a miss.

## What changed

| File | Change |
|---|---|
| `src/features/ai-extraction/photo-review-panel.tsx` | Read from photo: the enquiry photo is no longer read automatically. One enquiry photo opens directly; several show the photo list first. An unread photo opens in a large "turn the photo" step with Rotate left / Rotate right, Choose another and Read photo. A turned photo is saved as a new photo and that copy is read. Take a photo is shown on touch devices only. The first screen (upload and list) is a narrower modal. |
| `src/features/order-booking/booking-header-form.tsx` | New booking page: the From Enquiry / Direct radio buttons now sit right beside the "Order Booking Type" title. |
| `src/features/order-booking/header-detail-tab.tsx` | Order list: new **Manager Approval** column (Approved / Pending), **Approve** item in the kebab menu with a confirmation modal. |
| `src/features/order-booking/capabilities.ts` | New `approveBooking` flag. "Create PI" now needs an approved booking. Clone is offered for confirmed bookings only. |
| `src/features/order-booking/api.ts` | `managerApprovedAt` / `managerApprovedByName` on the booking types, new `approveOrderBooking` call. |
| `src/features/auth/permissions.ts` | New `ORDER_BOOKING_APPROVE` right. |
| `src/features/pi-status/new-pi/new-pi-page.tsx` | An order opened from Order Booking that is not approved yet is turned away with a clear message. |
| `src/features/pwa/order-booking/mobile-order-detail.tsx` | Phone order detail follows approval for "Create PI". Print Work Order is disabled for a draft work order. |
| `src/features/order-booking/work-order-print-menu.tsx` | Print Work Order is disabled for a draft work order (new `printWorkOrderBlocked`). |
| `src/features/order-booking/work-order-row-menu.tsx` | Same, in the Work Orders list kebab menu. |
| `src/features/pwa/work-orders/mobile-work-order-detail.tsx` | Same, in the phone work order detail. |
| `src/features/notifications/record-notifications-dialog.tsx` | Changes to the notification modal opened from PI Status. **Author to fill in the exact change here** (the assistant that drafted this file did not see this edit). |
| `tests/rbac-capabilities.test.ts` | Updated the "Create PI" case for approval, added cases for Approve and Clone. |

## Why

**Manager Approval.** The business wants a manager to sign off a confirmed order before a PI can be raised. It follows the existing PI Manager Approval step (migrations 173/174), so the pattern is familiar to reviewers.

**Read from photo.** Reading started with no way to correct a sideways photo, and an enquiry with several photos could not be chosen from. The AI reads the stored file exactly as it is, so the turn has to be saved before reading.

**Small fixes.** Clone on a draft, Print Work Order on a draft work order, and Take a photo on a desktop browser were offered but not useful. The radio buttons were far from the title they belong to.

The easy parts were the screen changes. The real work was the approval step (new right, migration, backfill rule, PI gate).

## How

**Manager approval**: a booking starts with the approval empty (Pending). Only a confirmed booking can be approved, by a user who holds `order_booking_approve`. The booking's status does not change. The New PI picker (`view=pi-picker`) now also requires an approved booking, and the "Create PI" shortcut follows the same rule.

**Backfill**: bookings that already have a PI are marked approved (approver empty, because nobody actually approved them). Confirmed bookings without a PI stay Pending. The backfill runs only the first time the column is added, so running the file again cannot approve a pending booking.

**Right**: added inactive in 181 and switched on in 182. No role is granted it; admins pass every check, other roles get it from Settings > Roles.

**Read from photo**: with exactly one enquiry photo the modal opens it directly, otherwise the user picks from the list. Rotating only changes the preview. On Read photo, a turned photo is uploaded as a new photo on the booking and that copy is read; an unturned photo is read as it is.

**Take a photo**: shown only when the device is touch-only (`(hover: none) and (pointer: coarse)`), not by window width. The camera code was kept, not deleted.

**Draft work orders**: Print Work Order is disabled with the tooltip "Work Order is still a draft". Print Scan Code keeps its existing rule (approved first).

**Kept the same on purpose**: the booking status values, the existing PI Manager Approval, the Print Scan Code rule.

## Decisions made

**The server does not block PI creation for an unapproved booking.** The New PI "Direct" flow creates, confirms and invoices a booking in one go, so a hard server block would break it. The gate is in the PI picker (server side), the Create PI shortcut and the New PI page. A hand-made request could still create a PI for an unapproved booking. Not picked: also gating Direct, because it needs a product decision.

**Backfill approves only bookings that already have a PI.** Approving every existing confirmed order was not picked because it would skip the new rule for orders still in flight.

**One approval step, one right.** The right is `order_booking_approve` (not a Manager role check), following the rule that business actions are given to roles as rights.

**Take a photo is hidden by device type, not removed.** Phones and tablets keep it; the same panel is also used by the Enquiry photo upload, so it is hidden there on desktop too.

**Draft print is blocked on screen only.** The server still accepts a print request for a draft.

## Bugs found while testing (already there before, not fixed here)

None.

## Notes

- Nobody except admins holds the new right yet. Which roles should get it is still open.
- The phone order list does not show the new Manager Approval column. The phone order detail does follow approval for "Create PI".
- Clone being confirmed-only also applies to the phone order detail, because both read the same rule.

## Next steps

- Decide which roles get `order_booking_approve`.
- Add the Manager Approval column to the phone order list.
- Set up the frontend test runner (`tsx` is not installed in the frontend, see Test plan).

## Test plan

**Automated tests:**
- [x] Frontend `npx tsc --noEmit -p .`: no errors.

**Manual checks (local):**
- [x] Read from photo: one enquiry photo opens directly and is not read until Read photo is clicked; several photos show the list first.
- [x] Rotate left / right before Read photo, Choose another, narrower first screen, Take a photo hidden on desktop.
- [x] New booking page: the radio buttons sit beside "Order Booking Type".
- [x] Order list: Manager Approval column, Approve in the kebab, confirmation, Create PI only after approval, Clone only on confirmed orders.
- [x] Print Work Order disabled for a draft work order.
- [x] PI Status notification modal change.
