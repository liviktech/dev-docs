# 2026-11-03 - forgot-password-rights-fixes (backend)

## Summary

There was no way to recover a forgotten password, a few screens returned 403 or 500 for users who only held the module they were working in, and the roles screen offered rights that should not be handed out. This PR adds a forgot password flow (OTP by email or WhatsApp, then a new password), lets the PI Status, FIR and Dispatch users do the actions that were blocked, fixes the SQL that broke gate passes, and renames the work order delete right to the production plan delete right.

Checked live against the dev database: migrations 115 and 116 ran, and the gate pass query now runs without errors. The full forgot password flow (real OTP going out through the notification service) was **not** checked end to end, see "Test plan".

**Removing the Production Line, Sales Visit and Proforma Invoice rights from the seed files is not included on purpose** - see "Decisions made" below. This was a choice, not a miss.

## What changed

| File | Change |
|---|---|
| `src/database/migrations/115_password_reset.sql` (new) | `core.password_reset` table. Holds the hashed OTP, its expiry, wrong attempts, and the hashed reset token. |
| `src/database/migrations/116_production_plan_delete_right_name.sql` (new) | Renames the display name of `work_order_delete` to "Production Plan - Delete". |
| `src/database/seeds/031_add_edit_rights.sql`, `032_fir_and_remaining_rights.sql` | Same rename, so a fresh database gets the new name. |
| `src/modules/auth/password-reset.service.ts` (new) | Request OTP, verify OTP, reset password. |
| `src/modules/auth/password-reset.repository.ts` (new) | SQL for the above: find the user by mobile, create and check the reset, queue the OTP message. |
| `src/modules/auth/auth.schema.ts`, `auth.routes.ts`, `auth.controller.ts` | New endpoints `POST /auth/forgot-password`, `/auth/verify-otp`, `/auth/reset-password` with validation. |
| `src/modules/auth/auth.service.ts`, `auth.types.ts` | Removed `isSuperAdmin` and `isAdmin` from the login response. Both only repeated `role`, and nothing in the apps needed them. |
| `src/modules/notifications/event-registry.ts` | New event `forgot_password_otp` (recipient type `USER`) so admins can switch its email and WhatsApp channels on and edit the wording on the Notifications page. |
| `src/docs/swagger/auth.swagger.ts`, `notification.swagger.ts` | Docs for the new endpoints and the new `USER` recipient type. |
| `src/modules/order-booking/order-booking.routes.ts` | Replica (Clone PI) and status (Submit) also accept PI Status Add. |
| `src/modules/dispatch/dispatch.routes.ts` | Creating a gate pass accepts dispatch Approve, Add or Edit. It needed Approve only. |
| `src/modules/dispatch/dispatch.repository.ts` | Fixed the gate pass SQL (see Bugs). |
| `src/modules/production-plan/production-plan.routes.ts` | Deleting a plan now needs the work order Delete right (now called Production Plan - Delete). |
| `src/shared/rbac/check_role_access.ts` | Added `fir` to the group of modules whose records can be viewed by a user holding any one of them, so FIR users can open the PI, work order and final invoice. |
| `.env.example`, `src/config/env.ts` | Access token default lifetime changed from 15m to 1h. This is not part of the work above, see Notes. |

## Why

Users could not reset their own password. Several users with a narrow module (PI Status only, FIR only, Dispatch only) hit 403 on actions the screen showed them. The gate pass SQL was simply broken. The rights list also needed tidying.

## How

**Forgot password**: the user enters a mobile number. The backend finds the login through the linked employee record (`hr.employee.mobile`, last 10 digits, must match exactly one active login). It makes a 6 digit OTP (valid 10 minutes, 5 wrong tries, 60 second gap between requests), stores only a hash, and queues one message per channel that is switched on for the `forgot_password_otp` event. Email goes to the login's email, WhatsApp to the mobile (a 10 digit number gets 91 in front). The wording comes from the company's rule or the default text. Verifying the OTP gives a one time reset token (stored hashed, valid 10 minutes). Setting the new password uses up the token and deletes every reset request of that user.

**Same answer for every number**: the forgot password endpoint answers the same whether or not the number belongs to a login, so it cannot be used to find accounts.

**Why a direct queue insert**: the OTP is only known at request time, so the message is rendered right away and written to `notification.message` with the text already filled in. The existing worker then delivers it like any other message. The event's `load` always skips, because there is no stored record for the worker to read.

**Permissions**: the Clone PI, Submit and gate pass routes use `requireAnyPermission` with the original right plus the right the calling screen already checks, so nothing was loosened for users who had neither.

## Decisions made

**Production Line, Sales Visit and Proforma Invoice rights were hidden in the roles screen (frontend) but not deleted here.** Seed files only run once and are already applied, so editing them would not change any database. Their rows are also spread through several seed files. Deleting the rows would break roles that already hold them, because the production scan, sales visit and proforma invoice routes still check those rights. A cleanup migration can be added once someone confirms no role depends on them.

**The work order Delete right was reused for production plans instead of adding a new right.** Nothing in the frontend deletes work orders, so the right was unused. A rename keeps existing role assignments. Anyone who could delete plans through Work Order Edit now needs this right.

**The login response lost `isSuperAdmin` and `isAdmin` instead of keeping them.** Every place that read them already checked `role` first. The server's own permission checks recompute the role from the database and never used these values.

## Bugs found while testing (already there before, not fixed here)

**1. Gate pass query was invalid SQL.** The "customer's own glass" part of `GATE_PASS_SELECT` had a misplaced bracket and mixed `json` with `jsonb` inside a `COALESCE`. Creating or opening any gate pass returned a 500 for every user. It came in with the customer material work (commit b386e73). **This one is fixed in this PR**, listed here so the cause is on record.

## Notes

- Nothing is sent until an admin switches on the "Forgot password OTP" channel (email, WhatsApp or both) on the Notifications page.
- Employees backfilled by migration 102 have the placeholder mobile `0000000000`. They need a real mobile saved before they can reset.
- The OTP text is stored in the notification message log, because the message is built when the OTP is made.
- `.env.example` and `env.ts` also change the access token default from 15m to 1h. That was not part of this work and is worth a second look before merging. The refresh token lifetime was left at 7d.
- Run `npm run migrate` for 115 and 116.

## Next steps

- Decide whether to delete the Production Line, Sales Visit and Proforma Invoice rights from the database with a migration, after checking which roles use them.

## Test plan

**Automated tests:**
- [x] Type check (`npx tsc --noEmit`): passes
- [ ] No new automated test was added for the password reset service or the changed permissions. Existing tests were not re-run for this PR.

**Manual checks (local):**
- [x] Ran migrations 115 and 116 against the dev database, both applied
- [x] Ran the gate pass query against the dev database with a dummy dispatch id, it returns 0 rows with no SQL error (it failed with a syntax error before the fix)
- [ ] Forgot password end to end (request OTP, receive it, verify, set new password, log in) - not run, needs the notification service and a channel switched on
- [ ] Create Gate Pass as a dispatch-only user in the app - not run, covered only by the query check above and the new route rule
- [ ] Open PI, work order and invoice from the FIR page as a FIR-only user - not run
- [ ] Clone PI and Submit as a PI Status only user - not run
