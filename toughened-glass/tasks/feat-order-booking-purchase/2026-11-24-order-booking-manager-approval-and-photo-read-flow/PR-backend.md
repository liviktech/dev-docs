# 2026-11-24 - order-booking-manager-approval-and-photo-read-flow (backend)

## Summary

Before this PR, any confirmed order could go straight to a Proforma Invoice (PI), nobody had to sign it off first. The "Read from photo" modal also read the enquiry photo by itself the moment it opened, with no way to turn a sideways photo first and no choice when an enquiry had several photos. A few screens also offered actions that did not make sense (Clone on drafts, Print Work Order on draft work orders, Take a photo on a desktop browser).

This PR adds a **Manager Approval** step for confirmed orders (backend and frontend), reworks the Read from photo flow, tidies four Order Booking / Work Order screens, and includes a change to the PI Status notification modal.

All changes were tested in the browser by the author. Automated results are listed in the Test plan.

**The server does not block PI creation for an unapproved order** - see "Decisions made" below. This was a choice, not a miss.

## What changed

| File | Change |
|---|---|
| `src/database/migrations/181_order_booking_approve_rights_catalog.sql` (new) | Adds the `order_booking_approve` right, inactive. |
| `src/database/migrations/182_order_booking_manager_approval.sql` (new) | Adds `manager_approved_at` / `manager_approved_by` to `sales.order_booking`, backfills bookings that already have a PI, switches the right on. |
| `src/shared/rbac/permissions.ts` | `ORDER_BOOKING_APPROVE` in the catalog. |
| `src/modules/order-booking/order-booking.routes.ts` | `PATCH /order-bookings/:id/approve`, checked with the new right. |
| `src/modules/order-booking/order-booking.controller.ts` | `approveBooking` handler. |
| `src/modules/order-booking/order-booking.service.ts` | `approveOrderBooking`; the booking and table rows carry the approval fields. Approving twice, or approving a booking that is not confirmed, is refused. |
| `src/modules/order-booking/order-booking.repository.ts` | `recordBookingManagerApproval`; the table query returns the approval date and approver name; the PI picker lists only approved confirmed orders. |
| `src/modules/order-booking/order-booking.types.ts` | Approval fields on the row and API types. |
| `tests/order-booking-route-rights.test.ts` | The new route must carry the new right. |
| `tests/rbac-permission-catalog.test.ts` | Migration 181 added to the catalog-versus-migration check. |

## Why

**Manager Approval.** The business wants a manager to sign off a confirmed order before a PI can be raised. It follows the existing PI Manager Approval step (migrations 173/174), so the pattern is familiar to reviewers.

**Read from photo.** Reading started with no way to correct a sideways photo, and an enquiry with several photos could not be chosen from. The AI reads the stored file exactly as it is, so the turn has to be saved before reading.

**Small fixes.** Clone on a draft, Print Work Order on a draft work order, and Take a photo on a desktop browser were offered but not useful. The radio buttons were far from the title they belong to.

The easy parts were the screen changes. The real work was the approval step (new right, migration, backfill rule, PI gate).

## How

**Take a photo**: shown only when the device is touch-only (`(hover: none) and (pointer: coarse)`), not by window width. The camera code was kept, not deleted.

**Draft work orders**: Print Work Order is disabled with the tooltip "Work Order is still a draft". Print Scan Code keeps its existing rule (approved first).

**Kept the same on purpose**: the booking status values, the existing PI Manager Approval, the Print Scan Code rule.

## Decisions made

**The server does not block PI creation for an unapproved booking.** The New PI "Direct" flow creates, confirms and invoices a booking in one go, so a hard server block would break it. The gate is in the PI picker (server side), the Create PI shortcut and the New PI page. A hand-made request could still create a PI for an unapproved booking. Not picked: also gating Direct, because it needs a product decision.

**Backfill approves only bookings that already have a PI.** Approving every existing confirmed order was not picked because it would skip the new rule for orders still in flight.

**One approval step, one right.** The right is `order_booking_approve` (not a Manager role check), following the rule that business actions are given to roles as rights.

**Reading a turned photo saves a new photo.** There is no server call to turn a stored photo in place. The original stays as it was.

**Draft print is blocked on screen only.** The server still accepts a print request for a draft.

## Bugs found while testing (already there before, not fixed here)

None.

## Notes

- Migrations 181 and 182 must be applied before the new backend is deployed, because the order list now selects the new columns.
- Nobody except admins holds the new right yet. Which roles should get it is still open.
- Clone being confirmed-only also applies to the phone order detail, because both read the same rule.
- The shared Neon database is not touched by this PR. Check which database you apply the migrations to.

## Next steps

- Decide which roles get `order_booking_approve`.
- Decide whether the New PI Direct flow should also wait for approval, and whether the server should then enforce it.

## Test plan

**Automated tests:**
- [x] Backend `tests/rbac-permission-catalog.test.ts` and `tests/order-booking-route-rights.test.ts`: 371 passing, including the new right matching migration 181 and the new route carrying the new right.
- [x] Backend `npx tsc --noEmit`: no errors.

**Manual checks (local):**
- [x] Print Work Order disabled for a draft work order.
- [ ] Migrations 181 and 182 on a scratch database - not run by the assistant (the DB test was blocked); apply and check on the database you use before merging.
