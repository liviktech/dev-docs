# 2026-10-17 - roles-rights-view-control (frontend)

## Summary

The screens ignored what rights a user had: someone with only View could still click every button, and someone with Control could lose buttons (such as View stages). This PR makes the screens follow one rule: **View shows the page and disables the action buttons (they stay visible); Control gives full access to the module.** A page the user has no right to open now shows a 403 Forbidden page instead of silently sending them to another module.

It covers Enquiries, Order Booking, PI Status, Work Orders (stages and pieces), Final Invoice and Dispatch on the web, plus the Sales Reps menu entry.

Checked: `tsc -b` and `vite build` pass. **Nothing was tried in a browser with real View-only or Control users**, see the test plan.

**The mobile (PWA) screens were not changed on purpose**, except the piece detail screen, so mobile still shows what it showed before.

## What changed

| File | Change |
|---|---|
| `src/features/auth/auth-context.tsx` | `canModule` now treats module level Control as covering every action. |
| `src/features/auth/guards.ts` | A user without the right is sent to `/forbidden` instead of `/`. |
| `src/routes/forbidden.tsx` (new) | The 403 Forbidden page, with a link back to the user's home page. |
| `src/components/layout/nav-items.ts`, `src/features/auth/accessible-routes.ts`, `src/routes/admin.sales-reps.tsx` | Sales Reps uses the new `master_sales_rep` module and the page has a View guard. It no longer opens for Enquiry rights. |
| `src/features/enquiries/enquiries-page.tsx`, `enquiry-row-menu.tsx` | View only: New Enquiry is disabled; the row menu shows Approve, Convert, Reopen and Delete disabled, and Edit reads View (opens read only). |
| `src/routes/orders.tsx` | New Booking always shows; disabled without Control. |
| `src/features/order-booking/header-detail-tab.tsx` | View only: Clone, Create PI and Delete disabled; Edit reads View. |
| `src/features/order-booking/products-tab.tsx`, `booking-header-form.tsx` | View only: the booking page is locked like a confirmed one (no Edit, Add Product, product changes, discount or Submit). |
| `src/features/pi-status/pi-status-page.tsx`, `components/pi-status-table.tsx` | View only: New PI disabled; Approve, Edit, Delete and Clone PI disabled in the row menu. |
| `src/routes/pi-status_.new.tsx` | The New PI page now needs Control, not only View. |
| `src/routes/work-orders.tsx`, `src/features/order-booking/work-order-row-menu.tsx` | Approve, Reject and View stages always show. Approve and Reject are disabled without Approve/Control. View stages is open to anyone with View. |
| `src/features/production-scan/scan-detail-page.tsx`, `src/features/pwa/work-orders/mobile-piece-detail.tsx` | Starting, completing and rejecting a piece need Work Order Control (or scanner Control). With View the pages open but those buttons are disabled; Reject piece is no longer hidden. |
| `src/features/final-invoices/*` | View only: Party/Admin Approve and Generate e-Invoice disabled; New Invoice disabled without Add/Control. |
| `src/features/dispatch/*` | View only: Approve Dispatch, Create Gate Pass and Generate e-Way Bill disabled. Existing gate passes and e-Way Bill download still open. |

## Why

Users reported that View access still allowed clicking everything, that Control access hid stage buttons, that Sales Reps showed up for Enquiry users, and that opening a module without rights landed on another page with no explanation. The buttons are disabled, not removed, so the user can see what exists and that it needs more access.

## How

`canModule(module, action, tab)` checks the exact right, then the module level right for tabs, then Control. Pages compute one flag (Control or the specific action) and pass `readOnly` or `disabled` down to the tables and menus. The route guard throws a redirect to `/forbidden`, so every protected route gets the 403 page without per page code.

Some pages already had a locked mode (a confirmed Order Booking, a closed Enquiry). View only access reuses it, so there is no second set of states to keep in step.

## Decisions made

**Disable instead of hide.** Asked for by the user: the buttons stay visible and greyed out.

**View is enough to open the Work Order stages.** Only the actions on them need Control. Scanner Control still works as an alternative so existing scanner operators are not locked out.

**e-Invoice and e-Way Bill keep their own rights.** They are separate modules, so a Final Invoice or Dispatch Control user also needs those rights to see Generate e-Invoice / e-Way Bill.

## Notes

- Needs the backend PR (new rights, `requireAnyPermission`, PI and scan routes) and `npm run seed` on the environment.
- The Control rule applies to every module, not only Work Order.
- Earlier changes in this branch that have no PR document yet are not described here: list state kept in the URL for Enquiry, Order Booking, PI Status, Final Invoice and Dispatch; a loader when switching tabs; the Approve Work Order dialog with delivery date, dispatch type and remarks; Edit removed from the Work Order menu.

## Next steps

- Apply the same View and Control rules to the mobile screens.
- Add Control checks to the backend routes that still only check login, so the API refuses what the screen disables.
- Gate the Sales Reps page's own Add and Edit buttons.

## Test plan

**Automated tests (no test files added or changed in this PR):**
- [x] `tsc -b`: passes.
- [x] `vite build`: passes.
- [ ] No automated tests cover the permission rules. None were added.

**Manual checks:**
- [ ] A View-only user on each module above: buttons visible but disabled, record details open - not tried.
- [ ] A Control user on each module: everything works, including Work Order stages - not tried.
- [ ] A user with no right opens `/orders` by URL: the 403 page shows - not tried.
- [ ] Sales Reps hidden for an Enquiry only user, shown with the new right - not tried.
