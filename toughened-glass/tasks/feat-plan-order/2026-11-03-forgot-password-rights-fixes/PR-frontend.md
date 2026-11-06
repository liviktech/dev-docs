# 2026-11-03 - forgot-password-rights-fixes (frontend)

## Summary

The login pages had no way to recover a forgotten password, the Production Plan and a few buttons were hidden or blocked for users who had the right module access, and the roles screen offered rights that should not be handed out. This PR adds the forgot password screens to both logins, makes the Production Plan follow the Work Orders rights with disabled (not hidden) buttons, lets PI Status and Dispatch users finish their flows, stops the open page from calling the API after logout, and tidies the roles screen.

Only the type check was run (passes). None of the screens were clicked through in a browser, see "Test plan".

## What changed

| File | Change |
|---|---|
| `src/features/auth/forgot-password-form.tsx` (new) | Three steps in one component: mobile number and Send OTP, then the OTP (with a 60 second resend timer), then new password and retype. "Back to sign in" sits at the bottom. |
| `src/features/auth/login-form.tsx`, `src/features/superadmin/login-form.tsx` | "Forgot password?" link that opens the form. After a successful reset it returns to sign in with a "Password changed" notice and empty fields. |
| `src/features/auth/api.ts`, `schema.ts` | API calls and Zod schemas for the three steps. Login response type no longer has `isSuperAdmin` and `isAdmin`. |
| `src/features/auth/accessible-routes.ts`, `guards.ts`, `auth-context.tsx`, `src/routes/m.index.tsx` | Removed the `isSuperAdmin` and `isAdmin` checks. They only repeated the `role` check next to them. |
| `src/features/auth/auth-context.tsx`, `src/lib/api-client.ts` | Logout now marks the app signed out, cancels the page's running queries and clears the user before calling `/auth/logout`. While signed out, `apiRequest` fails at once without a network call (login, logout and forgot password calls are exempt). Login clears the flag. |
| `src/routes/work-orders.tsx` | Production Plan tab and New Plan follow the Work Orders rights (View, Add). New Plan is always shown and disabled without Add. |
| `src/features/production-plan/production-plan-panel.tsx` | Edit, Delete and Approve are always listed for a plan and disabled with a reason when the user cannot use them. Before, a plan raised by someone else showed an empty Actions column. Delete follows the Production Plan - Delete right. |
| `src/features/production-plan/production-plan-editor.tsx` | Add Row, Save, Submit and Delete are shown and disabled without access. Add for a new plan, Edit for an existing one. |
| `src/features/order-booking/products-tab.tsx` | Submit is enabled for users with Order Booking Add or Edit, or PI Status Add. It now has a hover reason when disabled. Product add, edit and delete stay locked for PI Status only users. |
| `src/features/dispatch/components/dispatch-table.tsx`, `dispatch-page.tsx` | New `gatePassDisabled` prop. Create Gate Pass follows dispatch Approve, Add or Edit instead of Approve only. |
| `src/features/roles/rights-picker.tsx` | Hides the Production Line, Sales Visit and Proforma Invoice modules and the Reports Add, Reports Edit and Final Invoice Delete rights. The Work Order card is headed "Work Order and Production Plan". |

## Why

Users with a narrow module could see actions but not use them, or could not see them at all. A forgotten password could only be fixed by an admin. After logout the page kept calling the API and showed failed refresh calls in the network tab.

## How

**Forgot password**: the same form is reused on both logins. It talks to the new backend endpoints and keeps the reset token only in component state.

**Disabled, not removed**: where a button used to disappear without access, it now stays and is disabled, with a tooltip that says why. The one exception is the per-row delete button inside the plan editor and its Actions column, which still hide without access.

**Logout**: calls stop at the source (`apiRequest`), so nothing 401s and nothing tries a refresh. A page reload resets the flag.

**Roles screen**: the hidden modules and rights are filtered in the picker only. The rights still exist in the database.

## Decisions made

**Production Plan follows the Work Orders rights, not the separate Production Plan tab rights.** Approve still uses the Production Plan tab's Approve right.

**The hidden rights were filtered in the picker, not deleted.** Existing role assignments and the backend checks for those modules keep working. See the backend PR.

## Bugs found while testing (already there before, not fixed here)

**1. Lint errors in `auth-context.tsx` and `work-orders.tsx`.** `npx eslint` reports "only export components" errors in `auth-context.tsx`, and unused variable and effect warnings in `work-orders.tsx`. They were there before and are not touched by this change.

## Notes

- Needs the backend PR: the new auth endpoints, the relaxed permissions (Clone PI, Submit, gate pass) and the production plan delete right.
- The refresh token behaviour was left as it was.

## Test plan

**Automated tests:**
- [x] Type check (`npx tsc --noEmit -p tsconfig.app.json`): passes
- [ ] No automated tests were added or run for these screens.

**Manual checks (local):**
- [ ] Forgot password on both logins, all three steps - not run, needs the backend and a notification channel switched on
- [ ] Production Plan as a user with only Work Order Add, only Edit, and neither - not run
- [ ] Clone PI and Submit as a PI Status only user - not run
- [ ] Create Gate Pass as a dispatch-only user - not run
- [ ] Logout from a page with running queries, no calls after it in the network tab - not run
- [ ] Roles dialog shows the new heading and no longer lists the hidden rights - not run
