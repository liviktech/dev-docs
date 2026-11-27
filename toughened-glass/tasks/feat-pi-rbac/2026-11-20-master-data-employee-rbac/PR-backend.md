# 2026-11-20 - master-data-employee-rbac (backend)

## Summary

Master Data and the Employee module were still on the old shared rights. All Master Data screens used one pair (`master_data_view` / `master_data_control` / `master_data_delete`), and Employee used `employee_*` rights that the API read through an `attendance_*` alias, with Attendance and Payroll hidden behind a Control right nobody could be given from the Roles dialog. This PR moves both onto the rights catalog, the same way Order Booking, PI Status and Work Order were done. Every Master Data screen now has its own View / Add / Edit / Delete rights, and every Employee tab has its own rights. The API checks them on every write, no matter what the frontend shows.

I did not run the four new migrations (162 to 165). There are no database credentials in this environment, so they are checked only by the catalog test, which compares the migration rows to the code. See "Test plan".

**Master Data reads are not gated on purpose** - see "Decisions made" below. This was a choice, not a miss.

## What changed

| File | Change |
|---|---|
| `src/shared/rbac/permissions.ts` | New `MASTER_DATA` module with one group per screen (108 rights, 31 screens) and new `EMPLOYEE` module with 8 rights (Employees, Attendance, Payroll groups). |
| `src/database/migrations/162_master_data_rights_catalog.sql` (new) | Adds the 108 Master Data rights, inactive. Nothing existing is touched. |
| `src/database/migrations/163_master_data_rbac_cutover.sql` (new) | Backs up old assignments, maps holders of the old shared rights to the new per-screen rights, activates the new rights, retires the old ones, removes the old `master_data` and `sales_rep` rows in `core.right`. |
| `src/database/migrations/164_employee_rights_catalog.sql` (new) | Adds the 8 Employee rights. The four that already exist (`employee_view`, `_add`, `_edit`, `_delete`) keep their id and role assignments and only get metadata. |
| `src/database/migrations/165_employee_rbac_cutover.sql` (new) | Same cutover steps for Employee, including the old `attendance_*` rights. |
| `src/modules/master/master-rights.ts` (new) | Works out which right a Master Data write needs from the screen, the HTTP method and the path. Also holds the `MasterMount` type. |
| `src/modules/master/master.routes.ts` | Mounts every Master Data router from one list (`MASTER_MOUNTS`) with the screen that guards its writes. The old `requireMasterDataRightForMutations` call is gone. |
| `src/modules/master/bank-account/bank-account.routes.ts` | Writes need the Edit right of the owner (Party, Supplier or Company). The routes that only have an account id accept any of the three. |
| `src/modules/master/admin-configuration/admin-configuration.routes.ts` | The hard-coded `requireRole("SUPER_ADMIN", "ADMIN")` is replaced by the Admin Configuration Edit right, checked at the mount. |
| `src/middlewares/permission.middleware.ts` | Removed `requireMasterDataRightForMutations`, which nothing uses now. |
| `src/modules/attendance/attendance.routes.ts` | All 12 routes use `requireRight` with the Employee rights (see "How"). |
| `tests/master-route-rights.test.ts` (new) | Walks every mounted Master Data router and checks every write resolves to a catalog right. Also checks the action rules and that one screen's rights do not open another. |
| `tests/employee-route-rights.test.ts` (new) | Route table for the Employee API, and checks that each tab's rights are separate. |
| `tests/rbac-permission-catalog.test.ts` | Added `162_master_data_rights_catalog.sql` and `164_employee_rights_catalog.sql` to the migrations it compares with the catalog. |
| `src/modules/order-booking/order-booking.repository.ts` | no real change from this work. The clone-booking change in the same commit (a cloned booking becomes DIRECT and drops the enquiry link) is a teammate's, not part of this PR. |
| `src/modules/dispatch/dispatch.service.ts`, `dispatch-shipment.service.ts`, `dispatch.types.ts` | no real change from this work. The dispatch edits in the Employee commit are a teammate's. |

## Why

The old Master Data right was one switch for about 30 screens, so a role that could add a shift could also edit a tax profile. Employee had the opposite problem: payroll and attendance marking needed a Control right that the Roles dialog never offered, and the API only worked through an alias between two names. The approach already used for the other modules (a catalog, one migration to add rights, one to cut over, one test that walks the routes) fits both. Most of the work was the role mapping, so nobody loses access on day one.

## How

**Master Data, one guard per mount**: `master.routes.ts` names the screen once, where its router is mounted. `master-rights.ts` turns the request into the right: POST needs Add, PATCH or PUT needs Edit, a `/deactivate` path or a DELETE needs Delete. This avoids editing about 25 router files. Rights are stored as `master_data_<screen>_<action>`.

**Single-action screens**: Company Profile and Admin Configuration only have View and Edit. Project and Currency only have Add. User and Role only have View.

**Roles, users and rights stay admin only**: they are still guarded by `requireRoleForMutations("SUPER_ADMIN", "ADMIN")` in `master.routes.ts`. A role that could edit them could grant itself any right, so this is not a catalog right. The test skips these mounts on purpose.

**Inventory locations**: left alone. They already carry their own Inventory rights.

**Employee, one set per tab**: Employees uses `employee_view / add / edit / delete`. Attendance uses `employee_attendance_view` (daily and monthly reads) and `employee_attendance_edit` (save daily, clear a month). Payroll uses `employee_payroll_view` (read) and `employee_payroll_edit` (save, change record status). Toggling an employee's status needs Edit, as before.

**Role mapping in 163**: `master_data_view` gives every screen's View. `master_data_add`, `_edit` and `_delete` give that action on every screen plus View. `master_data_control` gives Add and Edit. `sales_rep_view` and `sales_rep_control` map to the Sales Rep screen. Writes to the company profile and to sales reps used to be limited by role code (MANAGER, CUSTOMER_CARE, SALES_MANAGER), so those new rights go only to holders in those roles. Admin Configuration Edit goes to nobody, because it was admin only.

**Role mapping in 165**: View, Add, Edit and Delete holders keep their right and also get Employee View and Attendance View. Only holders of the old Control right (reachable only through old role profiles) get attendance marking and payroll, because that is where those routes were allowed before. Nobody gains access.

**Every cutover backs up first**: assignments are copied to `core.rights_assignment_backup_163` and `_165` before anything changes, so the old state can be rebuilt.

## Decisions made

**Reads stay open to any signed-in user.** Other modules fill dropdowns from the Master Data lists (parties, products, glass types and so on). Gating GET would break them for roles without Master Data View. The View right only controls the Master Data screens, menu entries and URLs, which is the frontend's job.

**Deactivating now needs Delete, not Edit.** The screens already hid the Delete button behind the Delete right, but the old API let an Edit holder call `/deactivate` directly. The API now matches the screen. This is a real tightening, so it is called out here.

**Rights per screen, not per group.** I asked and the choice was per screen (about 100 rights) over per navigation group (about 25), for finer control. Glass Type, Glass Thickness, Glass Color and Glass Finish are separate screens, not part of Product Details.

**Users tab on the Employee page keeps its admin-only check.** I did not give it a catalog right. Login accounts are admin only in the API, and a right would have shown the tab to everyone who can view Master Data.

**The old `employee_*` to `attendance_*` alias was left in `check_role_access.ts`.** Nothing asks for `attendance_*` any more, but the sidebar and mobile menu still use that name for visibility, and removing it was outside this change.

## Notes

- Not all Master Data screens could be checked by hand. The route test covers every mounted router, including the nested payment modes router.
- `bank-account` routes that only have an account id accept the Edit right of Party, Supplier or Company. The account's real owner is not looked up.
- Notifications and Archive have their own modules and were not touched.

## Next steps

- Run migrations 162 to 165 on a copy of real data and check the role mapping, before running them anywhere shared.
- Remove the `employee_*` to `attendance_*` alias once the menu code stops using the `attendance` name.

## Test plan

**Automated tests (all new and changed tests passing, 6 old failures unrelated):**
- [x] `tests/master-route-rights.test.ts`: every write on every mounted Master Data router resolves to a catalog right. Also checks that POST gives Add, PATCH gives Edit, `/deactivate` gives Delete, that Company and Admin Configuration have a single write right, that one screen's rights do not open another, and that the old shared rights open nothing.
- [x] `tests/employee-route-rights.test.ts`: all 12 attendance routes need the expected right, Employees rights do not open Attendance or Payroll, and the old Control and `attendance_*` names open nothing.
- [x] `tests/rbac-permission-catalog.test.ts`: the catalog matches migrations 162 and 164, with no dependency cycle.
- [x] `npx tsc --noEmit`: clean.
- [x] Full backend suite after the Master Data change: 888 tests, 7 failing. 6 are the older purchase schema and PI reminder failures. 1 is `list-paging-schemas.test.ts`, which imports `dispatch-plan.schema.js`, a file that does not exist. None come from this work. I did not re-run the full suite after the Employee change, only the Employee and catalog tests.
- [ ] Lint: I ran ESLint on the touched folders but it printed only its configuration help text, so lint is not verified.

**Manual checks (local):**
- [ ] Migrations 162 to 165 - not run, no database credentials here. Covered only by the catalog test.
- [ ] Live API calls with a role that holds only some rights - not done.
