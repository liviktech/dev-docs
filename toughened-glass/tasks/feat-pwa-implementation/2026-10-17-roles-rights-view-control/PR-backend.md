# 2026-10-17 - roles-rights-view-control (backend)

## Summary

Users who were given View or Control rights on a module got "Access denied" errors on screens they were meant to use (for example PI Status, Work Order stages), because some APIs checked a different module than the screen is granted under. Control also did not cover every action.

This PR makes one rule hold everywhere: **Control on a module is full access to that module, View is read only**. It lets an API accept rights from more than one module, uses that on the Proforma Invoice and Production Scan routes, and adds the missing Sales Rep, Final Invoice and Dispatch rights to the database.

Checked: `tsc --noEmit` passes, and the seeds were applied to the local database. **Nothing was tried with real View-only or Control users**, see the test plan.

**The other routes were not reviewed on purpose** (see "Next steps"). Only the routes named below were changed.

## What changed

| File | Change |
|---|---|
| `src/middlewares/permission.middleware.ts` | New `requireAnyPermission([[module, action], ...])`. The request passes when the user holds any one of the listed rights. When none match, the error names the first one, as `requirePermission` does. |
| `src/modules/proforma-invoice/proforma-invoice.routes.ts` | Every route accepts the old `proforma_invoice` right **or** the `pi_status` one. List and view need View; create, edit, delete and status update need `pi_status` Control. |
| `src/modules/production-scan/production-scan.routes.ts` | Reading stages and pieces accepts `barcode_scanner` View or `work_order` View. Starting, completing and rejecting a stage accepts `barcode_scanner` Control or `work_order` Control. |
| `src/shared/rbac/check_role_access.ts` | `checkModuleActionAllowed` treats Control (module level or tab level) as covering every action (View, Add, Edit, Delete, Approve, Status update...). |
| `src/database/seeds/021_sales_rep_rbac.sql` (new) | Adds the `master_sales_rep` module with View and Control rights. |
| `src/database/seeds/022_final_invoice_dispatch_control_rbac.sql` (new) | Adds a Control right for `final_invoice` and `dispatch`. They only had View and Status update. |

## Why

- A user with PI Status rights got `missing required permission 'proforma_invoice_view'` and no data, because the PI API checked the `proforma_invoice` module while the screen is granted as `pi_status`.
- Work Order stage actions were blocked for a user with Work Order Control, because the scan API wanted the scanner module's rights.
- Sales Reps appeared in the menu for anyone with Enquiry rights, because the page was tied to the Enquiry module. A separate module is needed so it can be granted on its own.
- Final Invoice and Dispatch had no Control right to give, so a role could not be set to "full access" for them.

## How

`requireAnyPermission` calls the existing `checkModuleActionAllowed` for each option and passes on the first match. Super admin and admin still pass everything. Write routes of the PI API are tied to `pi_status` **Control** (the screen only has View and Control), while the old `proforma_invoice` rights keep working for anyone who already has them.

The Control rule is checked after the exact right and the tab to module fallback, so nothing that passed before stops passing.

## Decisions made

**Accept either module instead of renaming one.** Existing roles are already granted under the old names. Accepting both keeps them working without a data migration.

**Control covers every action, in one place.** It is done in the shared check, not route by route, so new routes follow the same rule.

**New module for Sales Rep instead of reusing a master one.** There was no suitable master right, and the page should be grantable alone.

## Notes

- Run `npm run seed` on every environment so seeds 021 and 022 exist. Without 021, the Sales Reps menu entry and page are hidden from everyone except admins, because the right to unlock them does not exist yet. Non admin roles that need the page must be given the new right in Roles & Access.
- On the local database, running the seeds also applied the pending 019 and 020 seeds.
- The Control rule now applies to every module, not only Work Order.

## Next steps

- Review the remaining routes (Enquiry create, edit and status, Order Booking writes, Final Invoice and Dispatch writes) so a View-only user cannot call them directly. The screens now disable the buttons, but only some APIs enforce it.

## Test plan

**Automated tests (no test files added or changed in this PR):**
- [x] Type check (`tsc --noEmit`) on the backend: passes.
- [ ] No automated tests cover the permission checks. None were added.

**Manual checks (local):**
- [x] Seeds 021 and 022 applied to the local database without errors.
- [ ] Log in as a user with only PI Status View: the list loads, create and edit are refused - not tried.
- [ ] Log in as a user with Work Order Control: start and complete a stage - not tried.
- [ ] Log in as a user with Work Order View only: stages can be read, completing is refused - not tried.
- [ ] Grant the new Sales Rep right to a role and check the menu entry - not tried.
