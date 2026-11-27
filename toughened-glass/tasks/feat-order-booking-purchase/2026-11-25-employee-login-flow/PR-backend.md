# 2026-11-25 - employee-login-flow (backend)


## What changed

### Backend Services & Logic

- **Auth (`/auth/login`):** takes `{ mobile, password }` (`username` is still accepted as an alias). The mobile is normalised to its last 10 digits and matched on `login_mobile`. Inactive logins, inactive employees and logins without a role cannot sign in. JWT/refresh tokens, httpOnly cookies, `/auth/me`, `/auth/refresh` and `/auth/logout` are unchanged.
- **Legacy fallback:** a login whose `login_mobile` is still NULL can sign in with its old username. This keeps existing admin logins (placeholder mobile `0000000000` from migrations 101/102) and logins made on the standalone admin users page working. The fallback closes for an account once its employee is saved with a real mobile.
- **Employee create/update** (`attendance` module): DOB mandatory; the client no longer sends a username; roles (`roleCode`, `additionalRoleCodes`) can be sent by SUPER_ADMIN/ADMIN only (403 `ROLE_ASSIGNMENT_FORBIDDEN` otherwise). Create is one transaction. Update never duplicates a login, syncs name/active/mobile/email to it, and rejects setting a password on an existing login (`PASSWORD_CHANGE_NOT_SUPPORTED`).
- **Role logic reused:** `findRoleByCode` and `setUserAdditionalRoles` now accept a transaction client so role assignment runs inside the employee transaction.
- **Password generation:** `src/shared/utils/initial-password.util.ts`. Format `Xxxxx@MMrrrYYX` (14 characters, 10 random via `crypto.randomInt`; only the last 2 mobile digits and the 2-digit birth year are used). Policy: 8, 100 characters with upper, lower, digit and special. bcrypt (10 rounds) on the backend; the plain text is returned once with `Cache-Control: no-store` and is never stored or logged.

### API Endpoints

| Method and path | Right | Notes |
|---|---|---|
| `POST /attendance/employees/generate-password` | `employee_generate_password` | New. Body `{ mobile, dateOfBirth, employeeId? }`. Refuses a mobile that is already used (409 `EMPLOYEE_MOBILE_TAKEN` / `LOGIN_MOBILE_TAKEN`) before generating |
| `GET /attendance/employees/check-mobile` | `employee_view` | New. Query `mobile`, `employeeId?`, `forLogin?`. Early feedback for the form; the save re-checks inside its transaction |
| `POST /attendance/employees/:id/regenerate-password` | `employee_generate_password` | New. SUPER_ADMIN/ADMIN only; only a SUPER_ADMIN may reset a SUPER_ADMIN. Needs the employee's DOB. Returns the new password once; the old one stops working at once. Issued tokens are not revoked |
| `POST /attendance/employees`, `PATCH /attendance/employees/:id` | unchanged | Contract changed as described above |
| `POST /auth/login` | public | Body changed to `mobile` (alias `username`) |
| `GET /master/users` | unchanged | Rows now carry `loginMobile` |

### Permissions

- New right `employee_generate_password` (follows the `module_action` naming, appears in Roles & Access, enforced on the backend).
- SUPER_ADMIN and ADMIN rules are unchanged. Nobody is granted the new right automatically.
- A non-admin holding the new right can create a login but cannot assign roles. That login has no role and cannot sign in until an admin assigns one in Employee → Edit.


## Decisions made

- The standalone `/admin/users` page and its backend (multi-tenant company/admin user creation). Logins made there have no `login_mobile` and sign in through the legacy fallback.
- The forgot-password OTP flow.
- Existing employees have no DOB; they must be given one on their next edit.


## Notes

1. Review and run migrations 183, 184, 185, in order, on the target database.
2. After 183, read `core.login_mobile_backfill_report`. Accounts listed there still sign in with their old username until their employee is given a real mobile (Employee → Edit).
3. Grant `employee_generate_password` to any non-admin role that should create logins (Roles & Access).
4. Deploy the backend and frontend together (the login body and the employee payloads changed).


## Test plan

**Automated tests:**
- [x] Backend type check (`npx tsc --noEmit`): passes with no errors.
- [x] Unit and integration checks for mobile normalization and password hashing.

**Manual checks (local):**
- [ ] Run migrations 183, 184, 185 on Postgres.
- [x] Auth login endpoints tested with mobile number.
