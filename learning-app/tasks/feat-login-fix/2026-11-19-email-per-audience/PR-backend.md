# 2026-11-19 - email-per-audience (backend)

## Summary

One email could only have one account in the whole `users` table. So if someone signed up in the admin panel with `grace@example.com`, the same email could not sign up in the mobile app, and the other way round. This PR lets one email hold one admin account and one mobile account (a student or a parent, never both). The two accounts are separate: own password, own sessions, own codes. Every email lookup now takes the audience (`admin` or `mobile`), so the admin panel and the mobile app never see each other's accounts.


## What changed

| File | Change |
|---|---|
| `src/database/migrations/0007_email-unique-per-audience.sql` (new) | Replaces the single unique index `users_email_key` with two partial unique indexes: `users_admin_email_key` (role `admin`) and `users_mobile_email_key` (roles `student` and `parent`). The new indexes are created first and the old one is dropped last, so the email is protected at every moment. |
| `src/modules/auth/interfaces/auth.interfaces.ts` | Adds the `Audience` type (`admin` or `mobile`), the `AUDIENCE_ROLES` map and the `isRoleInAudience` helper. |
| `src/modules/auth/repositories/users.repository.ts` | `findByEmail` and `findCredentialsByEmail` now take an audience and only look at roles of that audience. The duplicate check on create accepts both new index names. |
| `src/modules/auth/auth.service.ts` | Mobile OTP login, verify email, Google login and password login pass `'mobile'`. Admin login passes `'admin'`. `refresh` now needs an audience and rejects a token of the other audience as `INVALID_REFRESH_TOKEN`. The old `requiredRole` argument is gone. |
| `src/modules/auth/auth.controller.ts` | The mobile `POST /auth/refresh` passes `'mobile'`. |
| `architecture.md` | The `users` table row lists the two new indexes. Section 10b has a new "Audiences (migration 0007)" point. The signup flow names the new index. |
| `test/integration/auth-audience.int-spec.ts` (new) | Integration tests for the new rules (see Test plan). |

## Why

The admin panel and the mobile app are two different front doors. A person can be a student in the app and also an admin of the panel with the same email. The single unique index made that impossible. The real work was making sure the two accounts cannot mix: a lookup by email alone can now match two rows, so every lookup had to be told which door it serves. The easy part was the migration.

## How

**Audience from the role**: There is no new column. `admin` is the admin audience. `student` and `parent` are the mobile audience. The map lives in `auth.interfaces.ts`, in one place.

**Lookups take the audience**: `findByEmail` and `findCredentialsByEmail` add `role = ANY(...)` for the audience. With the two partial indexes, a lookup can match at most one row.

**Mobile only sees mobile accounts**: `/auth/request-otp`, `resend-otp`, `verify-email`, `login`, `google` and `refresh` only see mobile accounts. An admin-only email gets "no account, sign up". Admin endpoints only see admin accounts.

**Refresh tokens stay in their audience**: A refresh token of an admin is refused on the mobile route and the other way round. The answer is the same `INVALID_REFRESH_TOKEN` as for an unknown token and nothing is changed in the session.

**Signup conflicts**: Mobile signup conflicts only with another mobile account, so a parent cannot sign up if a student already has the email. Admin signup conflicts only with another admin. The conflict is still decided by the database index, not by a pre-check, so two signups at the same time still give one account.

**Kept the same on purpose**: Response shapes, error codes, rate limits and the OTP flow. The mobile app needs no change.

## Decisions made

**Two partial unique indexes instead of a unique index on email and role.** A plain `(email, role)` index would allow a student and a parent with the same email. The client wants one mobile account per email, so student and parent share one index.

**No new column.** The audience comes from the role, so no data needs to move and the migration has no data step.

**The old `POST /auth/login` now sees only mobile accounts.** It used to also accept an active admin with a password. Admins use `/admin/auth/login`, so this was closed on purpose.

**`refresh` has no default audience.** The old argument defaulted to "any role". Now the caller must say which door it serves, so a new route cannot forget it.

## Notes

- The down migration fails if one email already has both an admin and a mobile account. Remove one of them first. The file says so.
- Existing data cannot break the new indexes. They are looser than the old one.
- Admin signup, mobile signup and logins were not checked in a browser or against a real database by me.
- `src/modules/auth/auth-admin.controller.ts` is not in this diff. It already passed `'admin'` to `refresh`, and that argument now means the audience, so it needed no edit.

## Next steps

- Run `npm run migrate:up`, then `npm run test:integration`. Then try `migrate:down` and `migrate:up` once.
- Check in the admin panel and the mobile app that the same email can sign up in both.

## Test plan

**Automated tests (53 unit tests passing, integration not run):**
- [x] `npm run typecheck`: no errors.
- [x] `npm run lint`: no errors.
- [x] `npm test`: 7 files, 53 tests passed. These do not cover the new rules, because they need a database.
- [ ] `test/integration/auth-audience.int-spec.ts`, admin first: the same email can sign up for mobile and both accounts stay. An admin-only email gets 404 `ACCOUNT_NOT_FOUND` on `request-otp` and no code is sent. Mobile password login for an admin-only email gives 401 `INVALID_CREDENTIALS`. An admin refresh token gives 401 `INVALID_REFRESH_TOKEN` on the mobile route. Not run, because migration `0007` has not been applied by the developer.
- [ ] `test/integration/auth-audience.int-spec.ts`, mobile first: the same email can sign up for the admin panel. Mobile OTP login still returns the parent account. Admin login still returns the admin account. A mobile refresh token gives 401 `INVALID_REFRESH_TOKEN` on the admin route. Not run, same reason.
- [ ] `test/integration/auth-audience.int-spec.ts`, one mobile account per email: a parent is refused when a student has the email (409 `EMAIL_ALREADY_REGISTERED`), a student is refused when a parent has it, and a second admin is refused. Not run, same reason.

**Manual checks (local):**
- [ ] `migrate:up`, `migrate:down`, `migrate:up` on a real database - not run, the developer runs these.
- [ ] Same email in the admin panel and the mobile app - not checked, because the migration has not been applied.
