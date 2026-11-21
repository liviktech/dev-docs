# 2026-11-21 - parent-login-web (web)

## Summary

The backend and the mobile app already let a parent sign up, log in with an emailed code, link to a child and read the child's dashboard. The web app only knew students. A parent could register on web, but the login page had no way to say "I am a parent", and after login they were sent to the student Hub, where every call failed with 403. This PR adds the parent side to the web app: a Student or Parent switch on login, role based redirects, the parent pages (Dashboard, Results, Activity, Profile, Add child), and a page where a student approves or removes a parent. It uses the same endpoints and the same behaviour as the mobile app. No backend or mobile change was needed.

## What changed

| File | Change |
|---|---|
| `src/lib/auth/api.ts` | `requestLoginOtp` and `resendOtp` now take `{ email, role }` and send `role` when it is set. |
| `src/lib/auth/hooks.ts` | `useAfterAuth` sends parents to `/parent/dashboard` and clears cached `parent` queries. The mutation hooks follow the new function signatures (no code change in them). |
| `src/components/auth/role-switch.tsx` (new) | The Student or Parent tab switch used on the login page. |
| `src/components/auth/login-form.tsx` | Shows the switch, a parent title and subtitle, and passes `role` to the OTP call and to `/verify-email`. |
| `src/components/auth/verify-email-form.tsx` | Reads `role` from the address and passes it to resend. |
| `src/app/(auth)/signup/contact/page.tsx` | The redirect after signup now carries `role`, so a new parent lands on the Parent tab. |
| `src/components/auth/role-gate.tsx` (new) | Sends a signed in person with the wrong role to their own home. |
| `src/app/api/[...path]/route.ts` | The allowlist now includes `parent/children`, `parent/children/lookup`, `parent/children/:id`, `parent/children/:id/dashboard`, `student/parent-requests` and its approve and reject routes. |
| `src/proxy.ts` | The matcher now includes `/parent/:path*`, so the parent pages need a login. |
| `src/lib/parent/types.ts`, `api.ts`, `hooks.ts`, `format.ts` (new) | Parent types, endpoints, TanStack Query hooks and formatting helpers, ported from the mobile app. |
| `src/lib/parent/selected-child.tsx` (new) | A context that holds the parent's children and the chosen child, shared by Dashboard, Results and Activity. |
| `src/components/parent/parent-ui.tsx` (new) | Small shared pieces: `Stat`, `StatGrid`, `ChangeBadge`, `LinkStatusBadge`, `EmptyNote`, `Avatar`. |
| `src/components/parent/child-profile-section.tsx`, `learning-activity-section.tsx`, `academic-performance-section.tsx`, `progress-section.tsx` (new) | The dashboard sections, ported from mobile. |
| `src/components/parent/child-switcher.tsx`, `child-page.tsx` (new) | The child chips (only when there is more than one child) and the wrapper that handles loading, errors, "waiting for approval" and "no access". |
| `src/components/parent/parent-shell.tsx` (new) | The parent navigation (Dashboard, Results, Activity, Profile) on top of `AppShell`. |
| `src/components/app/app-shell.tsx` | `AppShell` now takes optional `nav` and `homeHref` props. The defaults are the old student values, so student pages look the same. |
| `src/components/app/user-menu.tsx` | The fallback name is "Parent" for parents, "Student" otherwise. |
| `src/app/(app)/layout.tsx` | Wrapped in `RoleGate`, so a parent who opens a student page goes to `/parent/dashboard`. |
| `src/app/(parent)/layout.tsx` (new) | `RoleGate` (parents only), the selected child context and the parent shell. |
| `src/app/(parent)/parent/dashboard/page.tsx`, `results/page.tsx`, `activity/page.tsx` (new) | The three child pages. |
| `src/app/(parent)/parent/profile/page.tsx` (new) | The parent's details, the list of children with remove or cancel (with a confirm dialog), Add a child, and Log out. |
| `src/app/(parent)/parent/add-child/page.tsx` (new) | Find a child by email, confirm who it is, then send the request. |
| `src/app/(app)/profile/parent-access/page.tsx` (new) | The student's page to approve, decline or remove a parent. |
| `src/app/(app)/profile/page.tsx` | Added a "Parent access" link card. |

## Why

Parents could create an account on web but could not use it. The backend (`/auth/request-otp` with `role`, the `/parent/*` routes, the `/student/parent-requests/*` routes) and the mobile screens were already done, so this PR is the web client and its proxy rules. The real work was keeping parents and students apart, since the proxy only checks that a login cookie exists, and building the dashboard pages so they match mobile.

## How

**Same data path as before**: the browser calls `/api/*`. The route handler keeps the tokens in httpOnly cookies and forwards to the backend. Only routes on the allowlist pass through. Token handling, refresh and logout are the same for every role, so none of that changed.

**Login role**: the switch sends `role` to `/auth/request-otp` and `/auth/resend-otp`. The backend answers "no account" when the account has the other role, and the page shows that message. After the code is verified, `useAfterAuth` reads `user.role` and picks the home page.

**Role gate**: `RoleGate` reads the current user and redirects when the role is not allowed. The student area allows `student` and `admin` and sends parents to `/parent/dashboard`. The parent area allows only `parent` and sends everyone else to `/hub`. If the user cannot be loaded (for example no network), the page still renders and the API stays the authority. The student `AnalyticsSession` now only mounts for students, because it sits inside the gate.

**Parent pages**: `ChildPage` is the one wrapper for Dashboard, Results and Activity. It picks the first approved child, shows the switcher when there are two or more, and covers all the empty and error cases the same way mobile does (no child yet, waiting for approval, 403 or 404 "no access"). The dashboard is refetched every time a page opens, and each page has a Refresh button instead of pull to refresh.

**Student side**: the student sees pending and approved parents, can approve or decline a pending one, and can remove an approved one after a confirm. Declined requests are hidden, as on mobile.

**Kept the same on purpose**: the backend, the API shapes, the signup pages (other than the redirect), the theme tokens, and the look of every existing student page.

## Decisions made

**Reused `AppShell` instead of copying it.** The parent area has the same sidebar and bottom tabs, so `AppShell` got `nav` and `homeHref` props with the student values as defaults. The parent nav lives in a client component (`parent-shell.tsx`) because icon components cannot be passed from a server layout.

**Client side role gate, not a proxy check.** The proxy only has the refresh cookie, which does not say the role. Reading the role would need an API call in the proxy on every page. The gate uses the cached `/auth/me` result, and the backend guards (`ParentRoleGuard`, `StudentRoleGuard`) are what actually protect the data.

**Refresh button, not pull to refresh.** Pull to refresh is a phone gesture. A button does the same job on desktop.

**Parent signup still asks the test questions.** The backend does not need `targetTest` and `testExperience` for parents, and mobile skips them, but the web signup pages already ask them in parent wording. This PR leaves that as it is.

## Bugs found while testing (already there before, not fixed here)

**1. `/plan` is not in the proxy matcher.** `src/proxy.ts` guards `/hub`, `/practice`, `/mock-exams`, `/stats`, `/profile` and `/onboarding`, but not `/plan`. A signed out visitor reaches the page and is sent to login only after the first API call fails. This was already there. It affects students only and no data is exposed.

## Notes

- Not checked in a live browser. The checks are type, lint and the existing unit tests. None of the new pages or the login switch have been looked at running.
- ESLint was run on the changed folders (`src/components/parent`, `src/components/auth`, `src/components/app`, `src/lib/parent`, `src/app/(parent)`, `src/app/(app)`), not on all of `src`. `next build` was not run.
- Google sign in is not tied to the Student or Parent switch. The backend does not take a role on `/auth/google`, so a parent who signs in with Google still ends up in the parent area through `useAfterAuth`.
- The backend and mobile app have no uncommitted changes for this feature (the parent work is already committed there), so there is no `PR-backend.md` or `PR-mobile.md`. The admin app has uncommitted spacing and tab changes that are not related to this work.

## Next steps

- Log in as a parent and as a student on desktop and on a phone width and walk the whole link flow: parent adds a child, student approves, parent sees the dashboard, student removes the parent.
- Add unit tests for `RoleSwitch`, `RoleGate` and the `useAfterAuth` redirect.
- Add `/plan` to the proxy matcher.

## Test plan

**Automated tests (48 passing):**
- [x] `npx vitest run`: all 9 existing test files pass. No new tests were added.
- [x] `npx tsc --noEmit`: no errors.
- [x] `npx eslint` on the changed folders: no errors.
- [ ] `next build` - not run.

**Manual checks (local):**
- [ ] Parent login: choose Parent, get a code, verify, land on `/parent/dashboard` - not run, login needs an emailed code.
- [ ] Choose Student with a parent email (and the other way round) and see the "no account" message - not run, same reason.
- [ ] Parent adds a child, student approves on Profile then Parent access, parent sees Dashboard, Results and Activity - not run, same reason.
- [ ] Student removes an approved parent and the parent sees "No access" - not run, same reason.
- [ ] A parent opening `/hub` is sent to `/parent/dashboard`, and a student opening `/parent/dashboard` is sent to `/hub` - not run, same reason.
- [ ] Parent pages at phone width (bottom tabs) and in dark mode - not run.
