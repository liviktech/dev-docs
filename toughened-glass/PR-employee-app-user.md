# PR: Employee as the main record, optional app-user login

## Summary
The employee is now the main record. A login (`core.app_user`: username, password, role, email) is created only when the employee is flagged as an **App user**, and is linked by a foreign key. The separate User master page is replaced by a Users sub-tab under Employees.

## Data model
| Table | Role |
|---|---|
| `hr.employee` | Employee details. New: `is_app_user BOOLEAN`, `user_id UUID` (FK to `core.app_user`, unique, `ON DELETE SET NULL`). |
| `core.app_user` | Login only. `role_code` is now **nullable**: a login created from an employee has no role until one is assigned in the Users tab. A role-less user cannot sign in (auth joins `core.role`). |

`core.app_user.full_name` is kept as a display mirror of the employee name (about 20 modules read it for created-by / approved-by). The employee is the source of truth and the name is synced on every employee create/update.

## Migrations (run `npm run migrate`)
- `101_employee_app_user.sql`: adds the two columns, unique index on `user_id`, makes `role_code` nullable.
- `102_backfill_employees_for_users.sql`: creates an `hr.employee` row for every existing login without one (`admin`, `superadmin`, ...), linked to it. Additive and idempotent; never modifies `core.app_user`. Backfilled employees get mobile `0000000000`, salary 0, and the next `EMP-nnn` code.

## Backend (`modules/attendance`, `modules/master/user`)
- Create employee: optional `isAppUser`, `username`, `password`, `email`. Username and password are required only when `isAppUser` is true. Employee and login are created in one transaction. Password is bcrypt-hashed.
- Update employee: ticking the flag creates a login; unticking removes it; username, email and password (blank keeps current) can be changed for an existing login. Duplicate username returns `USERNAME_TAKEN` (409).
- `DELETE /attendance/employees/:id` (attendance CONTROL): deletes the employee (attendance and payroll rows cascade) and its login. If other records still reference the login it is deactivated instead of deleted.
- Deactivating or reactivating an employee syncs the login's `is_active`.
- **Protected logins:** an employee linked to a `SUPER_ADMIN` or `ADMIN` login cannot be deleted, deactivated or unlinked (`PROTECTED_EMPLOYEE`, 400), so nobody can be locked out.
- `/master/users`: list queries use `LEFT JOIN core.role` so role-less users appear. User update no longer changes the name (comes from the employee). Auth, JWT and permission code are unchanged.

## Frontend
- Employee form: "App user (can log in)" checkbox; Email, Username and Password (with show/hide eye) appear when ticked. Editable in edit mode; blank password keeps the current one. Joining Date field is half width and the date picker / checkbox open only from their own label.
- Employees page: tabs are Employees, Attendance, Payroll. The Employees tab has sub-tabs **All Employees** and **Users**. The Users sub-tab lists logins (username, name, email, role or "No role assigned", status) with an **Assign Role** action that opens the existing user dialog (primary and additional roles).
- Row kebab menu: Edit and Delete (with confirmation).
- Roles & Access is a separate sidebar item under a "System" group (`/admin/roles`); its page was extracted to `features/roles/roles-access-page.tsx`.
- The User item was removed from the sidebar; the `/admin/users` route still exists.

## How to test
1. Run migrations; confirm `admin` and `superadmin` still log in with the same menus and module access.
2. Both appear in the employee list as app users; deleting, deactivating or unticking them is refused with a message.
3. Add an employee with App user ticked (username, password, email): one `hr.employee` row and one `core.app_user` row; the Users tab shows "No role assigned". Assign a role, then log in as that user.
4. Add an employee without the flag: no `core.app_user` row.
5. Edit: untick removes the login; change username, email, password; a duplicate username shows the conflict error.
6. Delete a normal employee: the row and its login are gone; attendance and payroll rows are removed.
7. `npx tsc --noEmit` passes in both repos (it did when this was written; the app itself was not run end to end).

## Known limitations
- The row status toggle shows a generic "Failed to change status" for protected logins; the edit and delete dialogs show the actual reason.
- Backfilled employees (`admin`, `superadmin`) appear in Attendance and Payroll with salary 0 until edited.
- `POST /master/users` direct creation still exists in the backend, but is no longer reachable from the sidebar.
