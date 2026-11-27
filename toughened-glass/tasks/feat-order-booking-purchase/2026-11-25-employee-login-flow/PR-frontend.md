# 2026-11-25 - employee-login-flow (frontend)

## Summary

The Employee form now owns the whole "person who can sign in" setup. One save creates the employee, the login, the primary role and the additional roles in a single transaction. Sign-in moves from a free-text username to the employee's **mobile number**. Passwords are generated on demand by an authorised user and stored only as a bcrypt hash.

| Before | After |
|---|---|
| Employee form: name, mobile, designation, salary, joining date; "App user" took a typed username and password | Adds a mandatory **date of birth**. "App user" has no username field (the mobile number is the username) and a manual **Generate Password** button |
| Roles assigned later from the Users tab ("Assign Role") | **Assign Role** is inside the Employee form (primary role mandatory, additional roles optional). The Users tab action is removed |
| Login screen asked for Username | Asks for **Mobile Number** |
| Editing an employee's mobile did not touch the login | A mobile change updates the login's username and login mobile (no second account) |

## What changed

- **Login screens** (normal and super-admin): "Mobile Number" field.
- **Employee dialog:** date of birth (mandatory), read-only "Username (Mobile Number)", Generate Password (manual only, gated by the right), read-only password field with show/hide and copy, Assign Role panel (primary + additional roles) for SUPER_ADMIN/ADMIN. For an existing login it shows the role and a **Regenerate Password** button for admins, with a confirmation.
- **Duplicate mobile handling:** the mobile field checks availability on blur; the error clears as soon as the number is edited; changing the mobile number or date of birth clears a generated password.
- **Users tab (desktop and mobile PWA):** Assign Role and the password-change UI are commented out (not deleted). The list now shows "Mobile (Login)".
- **Shared component:** `features/users/role-assignment-fields.tsx` extracted from the user dialog and reused.
- **Mobile PWA employee form:** updated to the same flow (DOB, generated password, role pickers for admins, regenerate).
- **Employee view dialog:** shows DOB, "Username (Mobile)" and Role.

## Decisions made

- The standalone `/admin/users` page and its backend (multi-tenant company/admin user creation). Logins made there have no `login_mobile` and sign in through the legacy fallback.
- The forgot-password OTP flow.
- Existing employees have no DOB; they must be given one on their next edit.

## Notes

- Access tokens already issued remain valid until they expire after a password regeneration or a user deactivation (`requireAuth` only verifies the JWT). This was already the case before this PR.
- The login mobile is unique across all companies, because the sign-in form has no company field.
- Changing the password of an existing login from the edit form is rejected; admins use Regenerate Password instead.

## Test plan

**Automated tests:**
- [x] Frontend type check (`npx tsc -b`): passes with no errors.

**Manual checks (local):**
- [x] All employee creation, mobile number sign-in, and password generation verified in the browser.
- The new and affected backend tests pass: `tests/employee-login-mobile.test.ts`, `tests/employee-route-rights.test.ts`, `tests/rbac-permission-catalog.test.ts`.
- The full `tests/` run has 10 failing tests in other modules (dashboard routes, sales visit route rights, Replace Order naming, new purchase schema, reminder slabs / PI reminders, list-paging schemas). None of them import the employee or login code in this PR. They are tracked separately.
