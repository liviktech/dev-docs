# 2026-11-21 - user-analytics (web)

## Summary

The web app (student-facing Next.js app) had no usage tracking. This PR adds a lightweight analytics client that starts a session on page load, sends heartbeats while the student is active, tracks screen views on key pages, and ends the session on logout. The tracking calls go to the backend's new `/analytics/*` endpoints (see `PR-backend.md`). No UI changes are visible to the student.

## What changed

| File | Change |
|---|---|
| `src/lib/analytics/tracker.ts` (new) | Low-level functions: `startAnalyticsSession`, `sendHeartbeat`, `endAnalyticsSession`, `trackEvent`. Handles the fetch calls to the backend's analytics client endpoints. `endAnalyticsSession` is best-effort (fires on logout, does not block if it fails). |
| `src/lib/analytics/hooks.tsx` (new) | `AnalyticsSession` component (starts a session on mount, heartbeats on an interval, ends on unmount) and `useTrackView` hook (fires a screen-view event once per mount). |
| `src/app/(app)/layout.tsx` | Renders `<AnalyticsSession />` inside the app shell so every signed-in page has an active session. |
| `src/app/(focus)/layout.tsx` | Same: renders `<AnalyticsSession />` in the focus (test-taking) layout so practice and mock exam screens are tracked too. |
| `src/app/(app)/hub/page.tsx` | Calls `useTrackView("dashboard_viewed")` to record when the student opens the home screen. |
| `src/app/(app)/plan/page.tsx` | Calls `useTrackView("study_plan_viewed")` to record when the student opens the study plan. |
| `src/app/(app)/stats/page.tsx` | Calls `useTrackView("statistics_viewed")` to record when the student opens the stats page. |
| `src/app/api/[...path]/route.ts` | Added the four analytics client paths (`analytics/sessions/start`, `analytics/sessions/heartbeat`, `analytics/sessions/end`, `analytics/events`) to the authenticated proxy list so the Next.js API route forwards them to the backend. |
| `src/lib/auth/hooks.ts` | Calls `endAnalyticsSession()` before `logout()` in the `useLogout` mutation so the session is closed before the token is revoked. Imported `endAnalyticsSession` from the tracker. |

## Why

The backend aggregation and admin reports need raw events and sessions from the student apps. Without client-side tracking, the analytics dashboard would show zero usage. The web app is the first client to send data.

## How

**Session lifecycle**: `AnalyticsSession` (a client component rendered in both layouts) calls `startAnalyticsSession` on mount, sets up a heartbeat interval, and calls `endAnalyticsSession` on unmount. The heartbeat keeps the session alive on the backend; if the student closes the tab without a clean unmount, the backend's scheduler expires the session after the timeout.

**Screen views**: `useTrackView` sends a single event per mount with the given event name. It runs in a `useEffect` with the event name as the dependency, so navigating away and back fires it again. Only three pages track views right now (hub, plan, stats); more can be added by calling the hook.

**Logout ordering**: the logout mutation calls `endAnalyticsSession()` first (best effort, at most a second), then revokes the token. If the session-end call fails or times out, the backend scheduler will expire it.

**Kept the same on purpose**: no existing UI, component, page behaviour or styling was changed. The student sees nothing different. The API proxy route only gained four new path patterns.

## Decisions made

**Tracking in both layouts.** The app layout (hub, plan, stats, profile) and the focus layout (practice, mock exams) are separate layout trees. Putting `AnalyticsSession` in both ensures every signed-in page is covered without a wrapper higher up that would also cover unauthenticated pages.

**Best-effort session end.** `endAnalyticsSession` does not throw if the call fails. If the tab is force-closed, the backend handles expiry. This avoids blocking the logout flow.

**Only three view events for now.** Adding `useTrackView` to every page at once would inflate the diff. The three busiest pages are tracked first; more can be added with one line each.

## Notes

- The tracking only works when the backend has migration `0016` applied and the scheduler is running (see `PR-backend.md`).
- The heartbeat interval is set in the `AnalyticsSession` component. The backend's `ANALYTICS_SESSION_TIMEOUT_MINUTES` (default 30) must be longer than the heartbeat interval for sessions to stay alive.
- The web app is on the `dev` branch, not `feat/user-stats`. The analytics changes were made directly on `dev`.

## Test plan

**Automated tests (none written):**
- [ ] No unit or integration tests were added for the analytics client. The tracker functions are thin fetch wrappers and the hooks are side-effect-only. The backend integration tests cover the endpoint behaviour.
- [ ] Type check: not run separately for the web app in this session.

**Manual checks (local):**
- [ ] Session start on page load, heartbeat calls in the network tab, session end on logout: not checked live. The backend endpoints are tested in `PR-backend.md`.
- [ ] Screen view events fired on hub, plan and stats pages: not checked live.
