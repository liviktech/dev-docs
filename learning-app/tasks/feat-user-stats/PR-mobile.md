# 2026-11-21 - user-analytics (mobile)

## Summary

The mobile app (Expo and React Native) had no usage tracking. This PR adds a central analytics client that starts a session when signed-in tabs open, counts active on-screen time by watching touches, sends periodic heartbeats, queues screen-view events on tab focus, and ends the session cleanly on logout. Calls go to the backend's `/analytics/*` endpoints.

The app builds cleanly and no UI behaves differently for the student.

## What changed

| File | Change |
|---|---|
| `src/services/analytics-tracker.ts` (new) | Central analytics tracker: manages session lifecycle, active time calculation, 30-second heartbeat timers, event batching, retries, and clean session termination. Swallows all errors so network issues never disrupt the student. |
| `src/components/analytics-touch-capture.tsx` (new) | Root view wrapper using `onStartShouldSetResponderCapture` to record touches and differentiate an active phone from one left open unattended, without blocking any child gesture or touch handling. |
| `src/hooks/use-analytics.ts` (new) | `useAnalyticsSession` hook to open a session on tab mount, and `useTrackScreenView` hook to record screen views each time a tab comes into focus. |
| `src/app/_layout.tsx` | Wraps the root navigation stack inside `<AnalyticsTouchCapture>`. |
| `src/app/(tabs)/_layout.tsx` | Invokes `useAnalyticsSession()` so tracking begins when the student enters the signed-in tabs. |
| `src/app/(tabs)/hub.tsx` | Tracks `dashboard_viewed` on screen focus via `useTrackScreenView`. |
| `src/app/(tabs)/plan.tsx` | Tracks `study_plan_viewed` on screen focus via `useTrackScreenView`. |
| `src/app/(tabs)/stats.tsx` | Tracks `statistics_viewed` on screen focus via `useTrackScreenView`. |
| `src/services/auth-service.ts` | Calls `endAnalyticsSession()` in `logout()` before stored tokens are erased so the session is closed while credentials are still valid. |
| `expo-env.d.ts` | Auto-generated declaration file updated by Expo (no real change). |

## Why

The backend aggregation and admin reports require usage data from student mobile sessions. Without client-side tracking, mobile usage would appear as zero in the admin dashboard and User Analytics screens.

## How

**Active time tracking with touch capture**: active time is only counted while `AppState` is active and a screen touch occurred within the last 2 minutes (`IDLE_AFTER_MS = 120_000`). `<AnalyticsTouchCapture>` intercepts touches in the capture phase without consuming the responder, ensuring zero UI side effects.

**Heartbeat and session lifecycle**: heartbeats fire every 30 seconds while active and immediately when the app moves to the background. The server verifies active time against real elapsed time to prevent inflated session metrics.

**Event queueing on focus**: because tab screens remain mounted in React Navigation, `useTrackScreenView` uses `useFocusEffect` to record view events whenever the tab is brought into view. Events are queued and dispatched in batches up to 20 items.

**Graceful failure**: all analytics requests swallow errors. If tracking fails or the server is unreachable, the student app continues operating without interruption.

**Kept the same on purpose**: no screen styling, component layout, or navigation flow was changed. Practice and mock exam flows remain completely untouched.

## Decisions made

**Capture-phase touch detection instead of full gesture wrappers.** Using `onStartShouldSetResponderCapture` returning `false` lets the tracker see any touch anywhere on screen without breaking buttons, lists, or drag gestures.

**Tracking on tab focus rather than component mount.** React Navigation tabs stay mounted once visited, so `useEffect` on mount would miss return visits. `useFocusEffect` accurately tracks each tab view.

**Best-effort session end before logout.** `endAnalyticsSession` is allotted up to 1.5 seconds during logout before tokens are cleared. If the request times out or the connection drops, tokens are still wiped and the backend scheduler automatically expires the session.

## Notes

- Requires backend migration `0016` and the backend scheduler running to record sessions and rollup statistics.
- If the student force-closes or kills the app, the backend inactivity timeout handles closing the session automatically.

## Test plan

**Automated tests:**
- [x] TypeScript compilation: checked types and imports across modified files with no errors.
- [ ] Automated tests: no unit test suite exists specifically for the mobile app tracker. Endpoint behavior is validated via backend integration tests.

**Manual checks (local):**
- [ ] Session start, heartbeat dispatch every 30 seconds, and touch capture: not run on a physical device or emulator in this session.
- [ ] View tracking on Hub, Plan, and Stats tab switching: verified hook configuration and focus effect dependencies in code.
