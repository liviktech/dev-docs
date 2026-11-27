# 2026-11-25 - employee-app-user-generate-password-assign-role-and-mobile-login (backend)

## Summary

Before this PR, a login was made in two steps: tick "App user" on the Employee form and type a username and password, then go to the Users tab to give that login a role. People signed in with that free-text username. Nothing generated a password, the employee had no date of birth, and changing an employee's mobile number did not touch their login.

This PR moves the whole setup into the Employee form (backend and frontend). One save now creates the employee, the login, the primary role and any additional roles in a single transaction. The mobile number becomes the username and the sign-in id. A **Generate Password** button, shown only to users who hold a new right, builds a password from random characters plus a few digits of the mobile number and birth year. Admins can **Regenerate Password** for an existing login. The Users tab loses its Assign Role action. The login screens ask for a Mobile Number.

It also closes a gap that was already there: an open session used to keep working after the user was deactivated or their password changed. Every token now carries a **token version**, and `requireAuth` checks it (and whether the login and employee are still active) on every request. A deactivation, password regeneration, admin password change or password reset ends the open sessions at once.

All changes were tested in the browser by the author. Automated results are listed in the Test plan.

**The migration files in this PR (183, 184, 185, 186) have not been run on the shared database or any environment** - see "Notes" below. The code does not work until they are applied.

**The standalone `/admin/users` page is not changed on purpose** - see "Decisions made" below. This was a choice, not a miss.

## What changed

| File | Change |
|---|---|
| `src/database/migrations/183_employee_dob_login_mobile.sql` (new) | Adds `hr.employee.date_of_birth` (nullable, for old rows) and `core.app_user.login_mobile` with a 10-digit check and a unique index across all companies. Back-fills `login_mobile` from the linked employee's mobile only when it is valid and not shared. Everything it could not link goes into the new table `core.login_mobile_backfill_report`. Never changes usernames or roles. Safe to run again. |
| `src/database/migrations/184_employee_generate_password_rights_catalog.sql` (new) | Adds the `employee_generate_password` right, inactive. |
| `src/database/migrations/185_employee_generate_password_cutover.sql` (new) | Switches the right on so it shows in Roles & Access. No role is granted it. |
| `src/database/migrations/186_app_user_token_version.sql` (new) | Adds `core.app_user.token_version` (integer, default 0). Existing users start at 0, so nobody is signed out by the deploy. Safe to run again. |
| `src/shared/rbac/permissions.ts` | `GENERATE_PASSWORD` action and `EMPLOYEE_GENERATE_PASSWORD` in the catalog. |
| `src/shared/utils/initial-password.util.ts` (new) | Builds the password and holds the password rules. |
| `src/modules/attendance/attendance.schema.ts` | Date of birth rules. Create and update take `dateOfBirth`, `password`, `roleCode`, `additionalRoleCodes` and no `username`. New schemas for generate-password and check-mobile. |
| `src/modules/attendance/attendance.types.ts` | Employee row carries `dateOfBirth`, `userId`, `loginMobile`, `roleCode`, `roleName`. |
| `src/modules/attendance/attendance.repository.ts` | Create is one transaction (employee, login, roles). Update syncs the login on a mobile change and never creates a second login. New mobile availability checks and the password regeneration queries. Deactivating an employee, switching a login off, and Regenerate Password raise the login's `token_version`. |
| `src/modules/attendance/attendance.service.ts` | Who may send roles or a password, generate-password with the duplicate check, check-mobile, regenerate password. |
| `src/modules/attendance/attendance.controller.ts` | Passes the signed-in user to the service. New handlers, with `Cache-Control: no-store` where a password is returned. |
| `src/modules/attendance/attendance.routes.ts` | New routes: `POST /employees/generate-password`, `GET /employees/check-mobile`, `POST /employees/:id/regenerate-password`. |
| `src/modules/master/role/role.repository.ts` | `findRoleByCode` can run on a transaction client. |
| `src/modules/master/user/user.repository.ts`, `user.types.ts`, `user.service.ts` | `setUserAdditionalRoles` can run on a transaction client. The users list returns `loginMobile`. Switching a user off or changing their password raises `token_version`. |
| `src/modules/auth/auth.schema.ts` | Login body takes `mobile` (`username` still accepted). |
| `src/modules/auth/auth.repository.ts`, `auth.types.ts` | Finds the login by mobile, with a fallback to the old username when `login_mobile` is empty. Also reads whether the employee is active and the user's `token_version`. New `findSessionState` (one primary-key read) for `requireAuth`. The token payload gets an optional `tv`. |
| `src/modules/auth/auth.service.ts` | Normalises the mobile number. Login, `/auth/me` and refresh also refuse an inactive employee. Tokens carry `tv`; refresh refuses a token whose version is no longer current. |
| `src/middlewares/auth.middleware.ts` | `requireAuth` now checks the user in the database after verifying the token: login active, employee active, token version current. Exposes `sessionLookup` so tests can supply the state without a database. |
| `src/modules/auth/password-reset.repository.ts` | A password reset raises the user's `token_version`. |
| `tests/employee-login-mobile.test.ts` (new) | Mobile normalising, login body, password format, employee schemas. |
| `tests/auth-session-state.test.ts` (new) | `requireAuth` refuses a deactivated login, an inactive employee, a missing user and an old token version; reads a token with no `tv` as version 0. |
| `tests/rbac-admin-gate.test.ts` | Supplies the session state through `sessionLookup`, because its made-up user has no database row and `requireAuth` now looks the user up. No change to what it asserts. |
| `tests/employee-route-rights.test.ts` | New routes added to the route-to-right table. The Employee right count is now 9. |
| `tests/rbac-permission-catalog.test.ts` | Migration 184 added to the catalog-versus-migration check. |
| `docs/employee-login-flow.md` (new) | The flow as built. |

## Why

The business wanted one place to create an employee, their login and their role, and the mobile number as the sign-in id, instead of a typed username that has to be remembered. Passwords had to be created by the system and never be guessable from the mobile or birth date alone, and never be stored in readable form.

The easy parts were the screens. The real work was the safe move to mobile login for logins that already exist, the single transaction for employee, login and roles, and the right so that only chosen people can create a login.

## How

**Password**: 14 characters in the pattern `Xxxxx@MMrrrYYX`: random capital, 4 random lowercase, random special, last 2 mobile digits, 3 random digits, last 2 birth-year digits, random capital. 10 of 14 are random (`crypto.randomInt`). It is made only when the button is clicked. The server saves only the bcrypt hash, sends the password back once, and never logs it.

**Roles in the same save**: the roles use the existing role checks and are applied inside the employee transaction, so a bad role leaves no employee and no login behind. Only Super Admin and Admin can send roles, as on the old Users tab. The last active Super Admin cannot be moved off that role.

**Duplicate mobile**: Generate Password and the mobile field both run the same checks as the save (employee mobile within the company, login mobile across all companies), so a password is never made for a number that cannot be saved. The save still checks again inside its transaction.

**Regenerate Password**: Super Admin and Admin only (only a Super Admin can reset a Super Admin). It needs the employee's date of birth. The old password stops working at once.

**Kept the same on purpose**: JWT and refresh cookies, tenant isolation, role-right resolution, the forgot-password OTP flow, and the Users list, search and status toggle.

## Decisions made

**Existing logins keep signing in with their old username until they get a real mobile.** Migrations 101/102 gave every existing login an employee row with the placeholder mobile `0000000000`, so a hard switch would lock out those admins. A login with no `login_mobile` can still use its username. This closes for an account as soon as its employee is saved with a real mobile. Not picked: forcing mobile login now.

**The new right is `employee_generate_password`, granted to nobody automatically.** Admins pass every check. Regenerate Password reuses the same right and is limited to admins in the service, so granting the right does not allow password resets.

**Editing cannot set a password on an existing login.** The server refuses it. Admins use Regenerate Password. A "view the original password later" option was not picked, because it needs a readable copy of the password.

**Role and rights changes do not raise the version.** They still reach the user when the access token is next refreshed, as before. Raising the version for them would force a new sign-in on every change. Decide this separately if rights should apply at once.

## Bugs found while testing (already there before, not fixed here)

None. (One bug that was already there, a deactivated user's open session staying alive until the token expired, is fixed in this PR; see "How".)

## Notes

- **Migrations 183, 184, 185 and 186 have not been run on the shared (Neon) database or on any environment.** Apply them in order before deploying the backend: the employee queries and the login now read the new columns. **186 matters most: `requireAuth` reads `token_version` on every request, so without it every authenticated request fails.**
- Which non-admin roles get `employee_generate_password` is still open.
- A company admin created on the admin users page has no employee row, so they never appear in the Employee list and sign in by username.
- Deploy the backend and frontend together: the login body and the employee payloads changed.
- The full backend run has 10 failing tests in other modules (dashboard routes, sales visit route rights, Replace Order naming, new purchase schema, reminder slabs, PI reminders, list-paging schemas). None import the code in this PR. The folder has no git baseline, so it is not proven that they failed before this work.

## Next steps

- Apply the four migrations (183 to 186) on the database you use, then review the backfill report.
- Decide which roles get `employee_generate_password`.
- Decide whether a role or rights change should also end open sessions (today it applies at the next refresh).
- Fix or retire the 10 failing tests in the other modules.

## Test plan

**Automated tests:**
- [x] Backend `tests/rbac-permission-catalog.test.ts`: passing, including the new right matching migration 184.
- [x] Backend `tests/auth-session-state.test.ts` (new, 9 cases) and `tests/rbac-admin-gate.test.ts`: 65 passing together.
- [x] Backend full run: 1162 tests, 1133 passing, 10 failing in unrelated modules (see "Notes").

**Manual checks (local):**
- [x] Sessions on a throwaway local database (migrations 1 to 186 applied there, the API run against it, then dropped): an employee session returns 200, then 401 on `/auth/me` and on refresh after an admin regenerates the password; the old password gets 401 and the new one 200; deactivating the employee gives 401, and re-enabling does not revive the old session while a fresh sign-in works; the admin's own session is unaffected; a token with no `tv` still works. The assistant ran this, not the author.
- [ ] Migrations 183, 184, 185 and 186 on the real database - not run; apply and check on the database you use before merging.
