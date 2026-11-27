# 2026-11-20 - master-data-employee-rbac (frontend)

## Summary

The Master Data screens all looked at one shared right pair, and the Employee screens went through an `attendance` name with a Control right that could not be granted. The backend now has its own rights for each Master Data screen and each Employee tab. This PR makes the screens use them, so buttons and menu entries are hidden when the user does not hold the matching right, and a restricted page cannot be opened by typing its URL.

I did not run the app in a browser. Type checks and the unit tests pass, but nothing was clicked through. See "Test plan".

**The Users tab on the Employee page still uses the admin check on purpose** - see "Decisions made". This was a choice, not a miss.

## What changed

| File | Change |
|---|---|
| `src/features/auth/permissions.ts` | Adds the 108 Master Data right codes and the 8 Employee right codes to the `Permission` list. |
| `src/features/auth/auth-context.tsx` | `canModule('master_x', action)` now looks up the exact right `master_data_x_<action>`. `CONTROL` is answered as Add or Edit. `sales_rep` is treated as a Master Data screen. The old rule that turned every `master_*` code into one shared right is gone. |
| `src/features/attendance/capabilities.ts` (new) | `getEmployeeCapabilities`, one flag per Employee right (same shape as the other modules). |
| `src/routes/attendance.tsx` | Route guard is now `createRightGuard(Permission.EMPLOYEE_VIEW)`. |
| `src/features/attendance/attendance-page.tsx` | Add, Edit and Delete come from the capabilities. The commented-out Attendance and Payroll tab code now refers to the new rights, ready for when those tabs come back. |
| `src/features/pwa/attendance/mobile-employees.tsx`, `mobile-attendance.tsx`, `src/routes/m.attendance.tsx` | Same change for the mobile screens. The mobile route checks `EMPLOYEE_VIEW` through `allowed`. |
| `src/routes/products.tsx`, `src/features/stock-masters/config.ts`, `src/features/glass-thickness/glass-thickness-page.tsx`, `src/features/pwa/master/heavy-adapters.tsx`, `src/features/auth/accessible-routes.ts` | Glass Types and Glass Thickness now use their own screens (`master_glass_type`, `master_glass_thickness`) instead of the Product right. The Products menu entry opens if any of the four glass tabs is allowed. |
| `src/components/layout/nav-items.ts`, `src/routes/admin.company.tsx`, `src/routes/admin.admin-configuration.tsx`, `src/features/admin-configuration/admin-configuration-page.tsx`, `src/features/pwa/master/mobile-company.tsx` | Company Profile and Admin Configuration use their own screens. They were both gated by Party before. |
| `src/features/inventory/components/purchase-payment-dialog.tsx`, `src/features/payment-details/payment-details-fields.tsx` | "Add company bank account" now needs Company Edit. Company has no Add right. |
| `src/features/items/item-dialog.tsx` | Adding a glass thickness from the item dialog needs Glass Thickness Add, not Product Add. |
| `src/features/items/items-page.tsx`, `src/routes/admin.sales-reps.tsx` | These pages used one Control flag for everything. Now Edit, Add and Delete are checked separately, so the Delete button needs the Delete right. |
| `tests/employee-capabilities.test.ts` (new) | Three tests for the Employee flags. |
| `src/features/roles/role-dialog.tsx`, `rights-picker.tsx` | no real change from this work. The Roles dialog edits in the same branch are a teammate's. `master_data` was already in the dialog's split-rights list. |
| `src/features/dispatch/*`, `src/features/pi-status/components/approve-party-modal.tsx` | no real change from this work. These came in the same commits and are a teammate's. |

## Why

Backend writes now need the right of the exact screen. If the screens kept checking one shared right, a user would see buttons that the API refuses, or lose buttons they are allowed to use. Doing this on the frontend also lets a role hold View on one screen without seeing the rest of Master Data.

## How

**One lookup, no implied rights**: `canModule` for a Master Data screen asks for the exact stored code (`master_data_shift_edit` and so on). Nothing is inferred from another right. The dependencies (Add needs View) are set in the database and enforced when a role is saved.

**Call sites kept**: the many `canModule('master_shift', 'ADD')` calls were not rewritten one by one. They keep working because the lookup changed in one place. Only the call sites that named the wrong screen were fixed (see the table).

**User and Role screens**: they only have a View right. Their pages ask for Control, which no longer maps to any right, so only SUPER_ADMIN and ADMIN get the buttons. That matches the API, which is admin only for writes.

**Employee**: the page and mobile screens read one set of flags from `getEmployeeCapabilities`. The Attendance and Payroll tabs are still switched off in the UI, so only the Employees tab is live. Their rights are checked by the API.

## Decisions made

**The Users tab on the Employee page still checks for an admin role.** It lists login accounts, which only SUPER_ADMIN and ADMIN can manage. A catalog right would have shown the tab to everyone who can view Master Data.

**The old nav and mobile menu still use the `attendance` module name.** It keeps working because the alias adds `attendance_*` for every `employee_*` right. Moving the menu to the new rights was out of scope.

**Master Data reads are not hidden.** The backend leaves GET open for dropdowns. The frontend only uses View to decide whether a screen, a menu entry or a URL is offered.

## Notes

- The Delete button on Items and Sales Reps now needs the Delete right. A role that had only Edit used to see it enabled and was refused by the API on deactivate. This is the visible side of the backend change.
- The mobile attendance daily screen (`mobile-attendance.tsx`) is not used anywhere. I updated it anyway so it does not point at a missing right.

## Next steps

- Click through Master Data and Employee with a role that holds only some rights, once the migrations are on a test database.
- Add a test for the new `canModule` lookup. It lives in the auth provider, so it was not covered by a unit test.

## Test plan

**Automated tests (new tests passing, 1 old failure unrelated):**
- [x] `tests/employee-capabilities.test.ts`: a user with no rights sees nothing, each right opens only its own control, and Payroll and Attendance are separate from the Employees tab. 3 passing.
- [x] Whole `tests/` folder: 74 passing, 1 failing. The failing one is `dispatch-plan-capabilities.test.ts`, which imports `features/dispatch-plan/capabilities.ts`, a file that does not exist. Not from this work.
- [x] `npx tsc -b`: clean.
- [ ] Lint: I ran ESLint on the touched folders but it printed only its configuration help text, so lint is not verified.

**Manual checks (local):**
- [ ] Master Data screens and the Employee page in a browser with a restricted role - not done.
- [ ] Opening a restricted URL directly (for example `/admin/shifts` without Shift View) - not done. The guard code is the same one the other modules use.
