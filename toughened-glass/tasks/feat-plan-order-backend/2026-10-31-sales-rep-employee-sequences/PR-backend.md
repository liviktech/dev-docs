# 2026-10-31 - sales-rep-employee-sequences (backend)

## Summary

Employee codes and sales rep codes were built in code (`EMP-001`, hand-typed rep codes) and ignored Master Data, Sequences. This PR adds `EMPLOYEE` and `SALES_REP` as sequence modules, numbers new employees and reps from them, and renumbers the existing ones when the sequence is created or changed. It also adds a `sales_rep` right, lets enquiry users see Sales Visits, lets the Employee module's `employee_*` rights work with the `attendance_*` checks, and fixes the internal error when a role code is edited.

Type check passes (`tsc --noEmit -p .`). Migrations 110 and 111 were run on the local database. The data steps listed under Decisions made were also run on the local database only.

**Other companies are not included on purpose** - see "Decisions made" below. Only the main company's existing reps and employees were renumbered. This was a choice, not a miss.

## What changed

| File | Change |
|---|---|
| `src/shared/utils/business-code.util.ts` | `EMPLOYEE` (prefix `EMP`) and `SALES_REP` (prefix `SR`) added, start 1, date format NONE. |
| `src/modules/master/sequence/sequence.types.ts`, `sequence.schema.ts` | The two new module names. |
| `src/modules/master/sequence/sequence.service.ts` | Definitions for the two modules. After a create or a real change (prefix, date format or number), existing employees or reps are renumbered. A save with nothing changed does nothing. |
| `src/modules/attendance/attendance.repository.ts` | New employees take their code from the Employee sequence when one is configured, else the old `EMP-001` way. New `recodeEmployeesFromSequence`. New `salesRepOnly` list filter (primary role or an extra role in `core.app_user_role`). |
| `src/modules/attendance/attendance.schema.ts` | `salesRepOnly` query option. |
| `src/modules/master/sales-rep/sales-rep.schema.ts`, `sales-rep.service.ts` | `code` is optional on create. When missing it is generated from the Sales Rep sequence. |
| `src/modules/master/sales-rep/sales-rep.repository.ts` | New `recodeSalesRepsFromSequence` (the Direct row is skipped). List is ordered by code. |
| `src/database/migrations/110_sales_visit_rep_cascade.sql` (new) | `sales.sales_visit` rep foreign key now cascades on update, so a rep recode carries sales visits along. Other tables already did. |
| `src/database/migrations/111_role_code_update_cascade.sql` (new) | The four foreign keys to `core.role(role_code)` now cascade on update, with their delete behaviour unchanged. |
| `src/database/seeds/030_sales_rep_module_rbac.sql` (new) | `sales_rep` module with view and control rights. Anyone with Master Data view or control keeps access to the Sales Reps screen. |
| `src/modules/master/role/role.repository.ts` | The role code is only updated when it actually changed. |
| `src/modules/auth/auth.service.ts` | Rights sent to the screens: enquiry users get `sales_visit_view`, and `employee_view` and `employee_control` also grant `attendance_view` and `attendance_control`. |
| `src/shared/rbac/check_role_access.ts` | The same two rules for the API checks. |

## Why

Business numbers are meant to be set in one place, and rep and employee numbers were the exception. The role edit failure came from changing a role code while other tables still pointed at the old one. The Employee screen showed "No modules assigned" because its rights are saved as `employee_*` while everything checked `attendance_*`.

## How

**Numbering**: new codes use the existing `generateBusinessCode`, so they follow the same date format and counter rules as every other module. A recode parks the old codes under temporary names first, then hands out new ones oldest first, in one transaction with a lock, so the unique code never clashes halfway.

**Rights as add-ons**: the Sales Visits and Employee rules are added when a user's rights are loaded. Nothing is written to a role, so they do not appear ticked in Roles & Access. Logging or editing visits still needs the Sales Visits control right.

## Decisions made

**Only the main company was renumbered and given sales rep logins.** The other companies have no sequence configured and look like test data. The same one-off can be run for them.

**Sales rep logins were created by a one-off script, not by code in the app.** Eight reps had no employee or user. Each got an employee, a login with a random password nobody knows (to be reset by hand) and the Sales Rep role. The script was deleted after running, so it is not in the diff.

**The rights are aliased instead of renamed.** Renaming `employee_*` to `attendance_*` (or the reverse) would touch stored role rights and every check. The alias fixes it with two small blocks.

## Bugs found while testing (already there before, not fixed here)

**1. Edits under `/master/*` are limited to a fixed list of role codes.** A Sales Rep role with the Sales Reps control right would see the buttons and get a permission error on save. Only view works for it today.

**2. Reps with the same name can be created.** Test saves made three reps called Santhoshh. Nothing blocks a repeat name.

**3. Right names are inconsistent.** The Employee module uses `employee_*` in Roles & Access and `attendance_*` in the checks. The alias hides it.

## Notes

- Existing codes are now `SR-20261031-001` to `-008` and `EMP-20261031-001` onward, because the saved sequences use the YYYYMMDD format. Set the date format to NONE and the number to 1 in Sequences to get `SR-001`. Saving renumbers again.
- Run `npm run seed` for the new `sales_rep` rights.
- The Direct sales rep row is still in use by enquiries. Whether to keep, hide or remove it is not decided.

## Next steps

- Decide on the Direct row (keep, hide, or remove and stop assigning it).
- Add the Sales Rep role to the master edit role list, or check `sales_rep_control` on the sales rep routes.
- Block repeat sales rep names.

## Test plan

**Automated tests (no test files cover these modules, none added):**
- [x] `tsc --noEmit -p .`: no errors.

**Manual checks (local):**
- [x] Ran migrations 110 and 111, both applied.
- [x] Recoded the main company's reps and employees and read them back from the database, codes are in order.
- [x] Ran the sales rep employee filter against the database, it returned the employee holding the role as an extra role.
- [ ] Created a rep and an employee through the app and checked the new codes. Not run, the app was not clicked through.
- [ ] Logged in as a Sales Rep user and as an Employee user to check the Sales Visits tab and the Employee module. Not run, only the rights code was changed.
- [ ] Edited and renamed a role from the screen. Not run in a browser.
