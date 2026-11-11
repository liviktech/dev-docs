# 2026-11-02 - add-edit-rights-rollout (backend)

## Summary

Routes checked the single Control right for most writes, and Master Data writes were limited by role code only. This PR makes the backend check Add, Edit and Delete per module to match the frontend, adds the Reports Download and Work Order Process rights, lets the order-to-dispatch modules read each other's records for the detail pop-ups, and adds seed files so the new rights appear in the role dialog. Attendance and payroll calculations were also reworked.

Everything was checked with the TypeScript compiler only. **The seeds were not run and no route was called with a non-admin login.**

## What changed

| File | Change |
|---|---|
| `src/shared/rbac/check_role_access.ts` | Add and Edit count as View. Control no longer covers Delete for the converted modules. Viewing any record in the Enquiry, Order Booking, PI Status, Work Order, Final Invoice and Dispatch flow is open to a user who holds any right in that flow. Enquiry Add and Edit carry over to Sales Visits. `employee_*` rights are mirrored as `attendance_*` for View, Add, Edit, Delete and Control. |
| `src/middlewares/permission.middleware.ts` | New `requireMasterDataRightForMutations` guard. |
| `src/modules/master/master.routes.ts` | Master Data writes now need Add, Edit or Delete on `master_data`. Roles, users, rights, companies and sales reps keep the old role check. Inventory locations carry their own check. |
| `src/modules/enquiry/enquiry.routes.ts` | Create needs Add, update needs Edit, close needs Delete, other status changes need Edit. |
| `src/modules/sales-visit/sales-visit.routes.ts` | Add, Edit and Delete. |
| `src/modules/inventory/inventory.routes.ts`, `src/modules/master/inventory-location/inventory-location.routes.ts` | Add, Edit and Delete on stock lots, edit requests and locations. These routes had no rights check before. |
| `src/modules/order-booking/order-booking.controller.ts` | Saving a new product with its process details and charges needs only Add. Editing a product still needs Edit. |
| `src/modules/proforma-invoice/proforma-invoice.routes.ts` | PI Status Control replaced by Add, Edit and Delete. |
| `src/modules/work-order/work-order.routes.ts`, `src/modules/production-scan/production-scan.routes.ts` | Stage updates and scanning need Work Order Process. |
| `src/modules/final-invoice/final-invoice.routes.ts`, `src/modules/dispatch/dispatch.routes.ts` | Edit instead of status update. Creating a gate pass needs Approve. |
| `src/modules/dispatch-plan/dispatch-plan.routes.ts` | Create needs Add, delete needs Delete. |
| `src/modules/fir/fir.routes.ts` | Add, Edit and Delete. |
| `src/modules/attendance/attendance.routes.ts` | Employee create needs Add, update and status toggle need Edit, delete needs Delete. New monthly attendance routes. |
| `src/modules/report/report.routes.ts` | The Sales Rep report export needs Reports Download. |
| `src/modules/auth/auth.service.ts`, `auth.types.ts` | The login response includes the role name. |
| `src/modules/attendance/attendance.service.ts`, `attendance.repository.ts`, `attendance.schema.ts`, `attendance.types.ts`, `attendance.controller.ts` | Monthly attendance list and clear. Payroll takes the month's attendance: zero present days means no salary, otherwise one day's pay is deducted per absent day (half day counts as half, company holidays are not deducted). |
| `src/database/seeds/031_add_edit_rights.sql` (new) | Add and Edit for every module that has a Control right, plus the first Delete rights. |
| `src/database/seeds/032_fir_and_remaining_rights.sql` (new) | FIR rights, and all the Delete and Process rights again, in case an earlier 031 had already run. |
| `src/database/seeds/033_reports_rights.sql` (new) | Reports View and Download. |
| `src/database/seeds/034_master_data_delete_right.sql` (new) | Master Data Add, Edit and Delete. |
| `src/database/seeds/035_employee_rights.sql` (new) | Employee View, Add, Edit and Delete. |

## Why

The frontend could only disable buttons, so the API still allowed a user with only Add to edit or delete. The role dialog also could not show the FIR, Reports and Employee rights because they were never in `core.rights`, which is the table it reads.

## How

**Rights check**: `checkModuleActionAllowed` accepts the direct right, then Control for everything except Approve, Discount and (for the converted modules) Delete, then Add or Edit for View.

**Delete**: admin-only unless the role holds `<module>_delete`. This applies to enquiry, sales_visit, inventory, order_booking, pi_status, work_order, final_invoice, dispatch, fir, master_data and attendance.

**Flow reads**: opening a work order, PI, booking or invoice from a number link needs a read on another module. Instead of changing every route, `VIEW` on any module in the flow is allowed for a user holding View, Add, Edit or Control on any module in the flow. This only affects reads.

**Master Data**: POST needs Add, PUT and PATCH need Edit, DELETE needs Delete. Deactivate is a PATCH, so it needs Edit.

**Seeds**: every insert uses `ON CONFLICT DO NOTHING`, so they only add rows. The seed runner skips files it has already run, so the later additions are in new files and not in 031.

## Decisions made

**Roles, users, rights, companies and sales reps keep the role-code check.** A Master Data Add right must not let someone create accounts or roles.

**Deactivate needs only Edit on the API.** It shares an endpoint with the Active/Inactive toggle, so the backend cannot tell a delete from a toggle. The frontend disables the Delete button without the delete right.

**Start of a work order still uses Approve.** Only stage start, stage complete and scanning use Work Order Process.

**Flow reads are broader than a single record.** The list endpoints open to the same users, not only the detail endpoints.

## Notes

- Verified with `tsc --noEmit`, which passes.
- Run `npm run seed` to create the new rights. Without it the role dialog does not list FIR, Reports, Employee or the Delete and Process rights.
- Roles that relied on Control for deleting, or for stage actions, need the new delete or process right.
- Users must log in again after their role changes.

## Next steps

- Check Master Data writes for the MANAGER, CUSTOMER_CARE and SALES_MANAGER roles. They used to write on role alone and now need Master Data rights.
- Decide whether the flow reads should be limited to single-record detail routes.
- Add rights checks to the sales rep, roles and users routes.

## Test plan

**Automated tests (no test suite was run):**
- [x] Type check (`npx tsc --noEmit`): passes.

**Manual checks (local):**
- [ ] Run `npm run seed` and confirm the new rights show in the role dialog - not done, the seeds were not run.
- [ ] Call a create, update and delete route as a user with only Add, only Edit and only View - not done, no non-admin login was available.
- [ ] Open a work order from a Final Invoice as a Final Invoice only user and confirm no 403 - not done.
- [ ] Create a payroll for a month and check zero-present and absent-day deductions - not done.
