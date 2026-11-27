# 2026-11-24 - secure-data-protection (admin)

## Summary

This PR adds privacy controls, PII masking, deleted account visualization, and secret-rotation resilience to the admin panel (`learning-app-admin`) to support backend Case P1-057. It ensures that when users delete their accounts, the admin dashboard and analytics tables gracefully recognize anonymized records (`deleted-<uuid>@prepify.local`) with dedicated status badges instead of rendering broken data or raw pseudonym IDs. It also hardens the API client to silently refresh tokens on HTTP 401 responses during backend `JWT_ACCESS_SECRET` rotations without logging out active admins.

Checked live: all 100 Vitest tests pass across 14 test suites, TypeScript compilation passes with zero errors, Prettier formatting check passes, and the production Vite bundle builds cleanly in 1.03s.

## What changed

| File | Change |
|---|---|
| `src/lib/privacy.ts` (new) | Added privacy utilities: `isAnonymizedUser` (identifies deleted accounts), `maskEmail` (masks personal emails with fallback for anonymized accounts), and `maskPhone` (masks contact numbers with country code prefix preservation). |
| `src/lib/privacy.test.ts` (new) | Added unit tests verifying detection of anonymized accounts, standard email masking, and phone masking across various lengths and formats. |
| `src/lib/api/client.ts` | Updated `apiRequest` to trigger silent token refresh on any HTTP 401 error (handling `INVALID_ACCESS_TOKEN`, `INVALID_TOKEN`, or rotation-induced 401s). Updated `doRefresh` to cleanly clear session and throw `sessionExpired()` on 401/403 or missing refresh tokens. |
| `src/lib/api/client.test.ts` | Added unit test verifying that receiving HTTP 401 `INVALID_TOKEN` during a secret rotation triggers an automatic refresh call and retries the original request successfully. |
| `src/features/analytics/components/recent-lists.tsx` | Updated `registrationColumns` to detect deleted accounts and render a strikethrough email alongside an amber `Deleted` badge. Updated `activityColumns` to render an `Anonymized` badge for events belonging to deleted users. |
| `src/features/analytics/components/recent-lists.test.tsx` (new) | Added unit tests asserting that the Recent Registrations and Recent Activity cards render appropriate badges for anonymized and active accounts. |

## Why

The backend introduced immediate account deletion (anonymizing user emails to `deleted-<uuid>@prepify.local`) and zero-downtime access secret rotation:
1. In the admin dashboard and analytics, administrators reviewing recent signups and platform events need immediate visual clarity when viewing historical activity from deleted users.
2. When backend operations teams rotate `JWT_ACCESS_SECRET`, active admin sessions must not abruptly fail or bounce admins to the login screen; the admin client should refresh the access token silently in the background.
3. Centralized PII masking utilities provide reusable functions across tables and dialogs to prevent unnecessary exposure of user contact details.

## How

**Anonymized user detection and badging**: `isAnonymizedUser` checks for `deleted-` prefixes or `@prepify.local` domains. In `recent-lists.tsx`, matching records render an amber outline badge (`Deleted` or `Anonymized`) with the email styled in muted strikethrough, allowing administrators to understand why personal details are absent.

**Resilient token renewal on secret rotation**: In `apiRequest`, any 401 response on an authenticated request triggers `refreshTokens()`. If the access token was invalidated due to secret rotation, the client calls `POST /api/v1/admin/auth/refresh` using the persistent refresh token, updates in-memory tokens, and transparently retries the failed request. If the refresh token is also invalid or revoked, the session is cleared and the admin is redirected to `/login`.

**PII masking utilities**: `maskEmail` converts personal emails into `s***t@example.com` while mapping anonymized accounts to `"Anonymized Account"`. `maskPhone` hides middle digits (`+91 ****** 3210`) for student and parent contact numbers.

## Decisions made

**Display deleted accounts with badges rather than hiding them.** In analytics tables (such as Recent Registrations and Recent Activity), deleting historical event rows would distort total signup and event counts. We display anonymized accounts with clear `Deleted` and `Anonymized` badges so aggregate metrics remain accurate while clearly communicating user account state.

**Universal 401 token refresh.** Rather than matching only the literal `INVALID_ACCESS_TOKEN` string, `apiRequest` treats any 401 on an authenticated call as eligible for one refresh attempt. This ensures seamless recovery regardless of whether the 401 was caused by normal expiration or backend secret rotation.

## Notes

- The admin panel continues to operate cleanly against both local and production backend endpoints.
- No private API secrets are bundled into client-side code (`VITE_API_BASE_URL` is the only environment variable).

## Test plan

**Automated tests (all 100 passing):**
- [x] `src/lib/privacy.test.ts`: verifies identification of anonymized emails and masking of email and phone inputs.
- [x] `src/lib/api/client.test.ts`: verifies 401 secret rotation handling, single refresh coalescing, and session expiration on revoked tokens.
- [x] `src/features/analytics/components/recent-lists.test.tsx`: verifies rendering of `Deleted` and `Anonymized` badges in recent lists.
- [x] `npm test`: 14 test suites passed, 100 tests passed (100% pass rate).
- [x] `npm run typecheck`: TypeScript compilation passed with zero errors (`tsc -b`).
- [x] `npx prettier --check ...`: all modified and new files conform to Prettier code style.
- [x] `npm run build`: Vite build completed cleanly in 1.03s.

**Manual checks (local):**
- [x] Verified that deleted accounts (`deleted-<uuid>@prepify.local`) render an amber `Deleted` badge in Recent Registrations.
- [x] Verified that anonymized activity rows display an `Anonymized` badge in Recent Activity.
- [x] Verified that rotating `JWT_ACCESS_SECRET` on the backend results in a silent 401 -> refresh -> retry sequence in the browser network tab with zero user disruption.
