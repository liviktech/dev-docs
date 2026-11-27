# 2026-11-02 - add-edit-rights-rollout (frontend)

## Summary

Access was a single Control right per module, so a user who could add could also edit and delete. This PR splits it into View, Add, Edit and Delete for Enquiry, Sales Visits, Inventory, Order Booking, PI Status, Work Order, Final Invoice, Dispatch, Dispatch Plan, FIR, Employee and Master Data, and adds a Reports Download right and a Work Order Process right. Buttons without the matching right now stay on screen and show disabled instead of disappearing. The Rights tab is gone and rights are picked inside the role dialog only. The Attendance and Payroll screens were rebuilt as monthly views, then switched off along with the Digital Signature section.

Everything was checked with the TypeScript compiler and ESLint only. **Nothing was clicked through in a browser with a non-admin login**, so the behaviour below is from the code, not from watching it run.

**Control is still checked on the screens that were not converted** (Roles, Users, Sales Reps, the mobile order screens and a few others) - see "Decisions made". This was a choice, not a miss.

## What changed

| File | Change |
|---|---|
| `src/features/auth/auth-context.tsx` | Add and Edit now count as View. Control no longer covers Delete for the converted modules (Delete is admin-only unless the role holds the module's delete right). New `isAdminRole` value. |
| `src/features/auth/rights-alias.ts` (new) | Treats each `employee_*` right as its `attendance_*` twin, because the Employee screens ask for `attendance_*`. |
| `src/features/auth/guards.ts` | Route guard accepts Add or Edit as View, lets Control cover Add and Edit, uses the Employee alias. |
| `src/features/auth/accessible-routes.ts` | Landing page logic accepts Add or Edit as View, uses the Employee alias, and now lists Final Invoice and Dispatch (a user with only those saw "No modules assigned"). |
| `src/features/roles/roles-access-page.tsx` | Rights tab removed. The rights code is kept in the file but unused. |
| `src/features/roles/rights-picker.tsx` | Control rights are no longer listed. |
| `src/features/roles/role-dialog.tsx` | Saving a role drops a leftover Control right for the converted modules, so it cannot switch Add and Edit on together. |
| `src/features/users/user-dialog.tsx` | Digital Signature section and its handlers commented out. The existing signature is still sent on save. |
| `src/features/auth/api.ts`, `src/components/layout/app-sidebar.tsx` | Sidebar shows the role name instead of the role code. |
| `src/features/attendance/attendance-page.tsx` | Two top tabs, Employees and Users. Users is for admins only. New Employee moved into the page header. Add, Edit and Delete rights on the Employee tab. Attendance and Payroll tabs commented out. |
| `src/features/attendance/attendance-modal.tsx` (new), `payroll-modal.tsx` (new), `daily-attendance-tab.tsx`, `payroll-tab.tsx`, `use-attendance-sheet.ts`, `api.ts`, `queries.ts`, `types.ts` | Monthly attendance table with an Add Attendance modal, and a payroll table with a Create Payroll modal. Month and date pickers sit in the modal headers. Still in the code but switched off. |
| `src/features/enquiries/enquiries-page.tsx`, `enquiry-row-menu.tsx` | Add, Edit and Delete rights. Close is the delete action and is disabled without the delete right. |
| `src/features/sales-visits/sales-visits-page.tsx` | Add, Edit and Delete rights. |
| `src/routes/inventory.tsx`, `src/features/inventory/stock-tab.tsx`, `purchase-tab.tsx`, `locations-tab.tsx`, `src/features/pwa/inventory/*` | Add, Edit and Delete rights. Edit Requests needs Edit. |
| `src/routes/orders.tsx`, `orders_.new.tsx`, `src/features/order-booking/header-detail-tab.tsx`, `products-tab.tsx`, `booking-header-form.tsx` | Add, Edit and Delete rights for bookings and for products inside a booking. |
| `src/features/pi-status/pi-status-page.tsx`, `components/pi-status-table.tsx`, `src/routes/pi-status_.new.tsx` | Add, Edit and Delete rights. The Change Requests button needs Edit. Create PI from a booking needs Add. |
| `src/routes/work-orders.tsx`, `src/features/production-scan/scan-detail-page.tsx`, `src/features/pwa/work-orders/*` | New Work Order Process right for stage actions and scanning. |
| `src/features/final-invoices/final-invoices-page.tsx` | Add and Edit rights. |
| `src/features/dispatch/dispatch-page.tsx`, `components/dispatch-table.tsx` | Edit right. Create Gate Pass needs Approve. |
| `src/features/dispatch-plan/dispatch-plan-tab.tsx`, `dispatch-plan-panel.tsx` | Add to create, Edit to change, Delete to remove. |
| `src/features/fir/fir-page.tsx` | Add, Edit and Delete rights. |
| `src/features/reports/components/report-export-button.tsx` | Every export and download button needs the Reports Download right. |
| `src/routes/admin.*.tsx` (charges, invoice terms, order charges, payment terms, sequences, shifts, tax profiles, work centers), `src/routes/party.tsx`, `src/features/products/product-details-tab.tsx`, `src/features/stock-masters/stock-master-page.tsx`, `src/features/glass-thickness/glass-thickness-page.tsx`, `src/features/company/company-page.tsx`, `src/features/shifts/shift-select.tsx` | Master Data screens use Add, Edit and Delete. |
| `src/routes/admin.users.tsx`, `src/routes/admin.sales-reps.tsx` | no real change |

## Why

Roles needed to give someone the ability to create records without letting them change or remove existing ones, and the other way round. The Control right could not express that. Delete is kept to admins by default because it is the hardest action to undo.

## How

**Rights model**: a module's right names are `<module>_view`, `_add`, `_edit` and `_delete`. `canModule(module, 'ADD')` and the other actions are checked directly. Control still covers View, Add and Edit when a role has it, so screens that were not converted keep working.

**Add and Edit count as View**: a role with only Add or Edit can open the module. The route guard and the landing page logic do the same check, because they read the rights list on their own.

**Delete**: for Enquiry, Sales Visits, Inventory, Order Booking, PI Status, Work Order, Final Invoice, Dispatch, FIR, Employee and the Master Data screens, Control does not cover Delete. Only SUPER_ADMIN, ADMIN or a role holding that module's delete right can delete.

**Disabled, not hidden**: actions without the right render as disabled so the user can see they exist. Things that depend on the record's state (an approved plan, someone else's plan, the Users tab for non-admins) are still hidden.

**Work Order Process**: starting and completing stages and scanning pieces check `work_order_process`. Approving a work order is still the Approve right.

**Reports**: all the export and download buttons share one component, so the Download check is in one place.

## Decisions made

**Control hidden in the role dialog, not removed.** Roles that already have Control keep working, and the screens that still check Control keep working. Removing it everywhere at once would lock users out.

**Roles, Users, Sales Reps and the mobile order screens were not converted.** Their backend writes are limited by role code on purpose, so a Master Data Add right cannot create accounts or roles.

**Employee code link stays plain text when the user cannot edit.** Only the buttons were changed to disabled.

**Mobile screens still hide some buttons** (for example the add buttons on stock and locations). Only the desktop screens were changed to disabled.

**The role dialog drops Control only for the converted modules.** Control on the other modules is kept when a role is saved.

## Notes

- Verified with `tsc --noEmit -p tsconfig.app.json` and ESLint. The only compiler errors are the existing ones in `src/features/superadmin/company-dialog.tsx` (zodResolver typing), which this PR does not touch.
- ESLint still reports existing fast refresh and setState-in-effect errors in several files this PR touches. They were there before.
- A user must log out and back in after their role changes, so the app reloads the rights.
- The Order Booking freeze after login for a non-admin user was never found in the code, so it is not claimed as fixed. The 403 on the order list was fixed in the backend PR.

## Next steps

- Convert the remaining screens (Roles, Users, Sales Reps, company, the mobile order screens) to Add, Edit and Delete.
- Change the mobile screens to show disabled buttons.
- Run through each module with a non-admin user holding only View, only Add and only Edit.

## Test plan

**Automated tests (no test suite was run):**
- [x] Type check (`tsc --noEmit -p tsconfig.app.json`): passes apart from the existing `company-dialog.tsx` errors.
- [x] ESLint on the edited attendance, roles and auth files: no new errors.

**Manual checks (local):**
- [ ] Log in as a user with only Add, only Edit and only View on each converted module - not done, no non-admin login was available. Needs a reviewer.
- [ ] Open a role in Roles & Access, save it, and confirm a leftover Control right is dropped - not done.
- [ ] Log in as a Final Invoice only user and confirm the page opens instead of "No modules assigned" - not done.
- [ ] Open a work order number from a Final Invoice as a Final Invoice only user - not done, depends on the backend PR.
