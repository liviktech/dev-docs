# 2026-11-20 - home-and-stats-dynamic (mobile)

## Summary

The Hub, Stats and Profile screens showed made-up data (fake streak, fake XP and "Top 8%", fake test history, fake subjects). This PR loads all three from the backend (`PR-backend.md`) and deletes the mock data and the study context that held it. The Hub shows the student's name, streak, target exam, quick access to practice, recent activity, and a mock exam card that is a placeholder with static data. Stats shows real totals, history, subject accuracy and Incorrect and Bookmarked lists. Profile shows real name, exam, streak, accuracy and questions answered.

`npx tsc --noEmit` and `npx eslint src` are clean. I did not run the app on a device or emulator, so none of the screens were looked at.

## What changed

| File | Change |
|---|---|
| `src/services/home-api.ts` (new), `src/hooks/use-home.ts` (new) | `getHome` and `useHome` for `GET /student/home`. Query key starts with `student`. |
| `src/services/stats-api.ts` (new), `src/hooks/use-stats.ts` (new) | `getStats`, `getReviewQuestions`, `useStats` and `useReviewQuestions` (infinite query, 10 per page). |
| `src/services/mock-exam-api.ts` (new) | Placeholder: static `getMockExamHighlight` and `getLatestMockScore` (returns null). Only these function bodies change when the real mock exam API exists. |
| `src/hooks/use-mock-exam-highlight.ts` (new), `src/hooks/use-latest-mock-score.ts` (new) | Hooks over the placeholder. The screens only talk to the hooks. |
| `src/hooks/use-quick-session.ts` (new) | Starts a 20 question practice session or opens the unfinished one. Shared by the Hub and Tests tab. |
| `src/utils/stats-format.ts` (new) | `accuracyOf` and `subjectColor`. |
| `src/components/home/section-header.tsx` (new), `src/components/home/mock-exam-card.tsx` (new) | Section title row and the mock exam card (disabled when it has no route). |
| `src/components/profile/stat-tile.tsx` (new) | One number tile on the Profile progress grid. |
| `src/app/(tabs)/hub.tsx` | Rewritten on real data: loading and error states, welcome and streak, focus card, Quick Access, Mock Exam section, Recent Activity. Pull to refresh and refetch on focus. |
| `src/components/hub/focus-card.tsx` | Takes the real exam and days remaining. The fake "mastered" bar is removed. |
| `src/app/(tabs)/tests.tsx` | Uses `useQuickSession` instead of its own copy of the start logic. |
| `src/app/(tabs)/stats.tsx` | Rewritten on `useStats`. History rows open that session's result screen. Pull to refresh and refetch on focus. |
| `src/components/dashboard/subject-performance.tsx`, `test-history.tsx`, `question-lists.tsx` | Take real data. Empty states added. Question lists load a page at a time with "Show more". |
| `src/app/(tabs)/profile.tsx` | Name, grade, school, exam and target, subjects count, streak, accuracy and questions answered are real. Latest mock score shows a dash. The fake "On track" chip and "Current focus" line are removed. The Target card shows only when there is an exam. |
| `src/hooks/use-practice.ts` | Submitting a session and toggling a bookmark also refresh the Home and Stats data. Starting a session refreshes Home. |
| `src/hooks/use-google-sign-in.ts`, `src/app/(auth)/verify-email.tsx`, `src/app/_layout.tsx` | Removed the study context calls and provider. No other change. |
| `src/components/hub/commitment-picker.tsx`, `copilot-card.tsx`, `module-card.tsx` (deleted) | Hub widgets with no real data behind them. |
| `src/components/home/stat-tile.tsx` (deleted) | Nothing used it. |
| `src/data/mock-hub.ts`, `mock-dashboard.ts`, `mock-exams.ts` (deleted) | The mock data. |
| `src/state/study-context.tsx` (deleted) | In-memory mock exam and subjects. Replaced by the API and onboarding data. |

## Why

The practice tests already saved real data, but the three main tabs ignored it. The goal was a Hub that opens practice quickly and shows recent activity, and Stats and Profile that show real progress. Mock exams do not exist yet, so the Hub needed a placeholder that can be swapped for the real API without changing the screen.

## How

**One Hub call**: the Hub, and the exam and streak on Profile, read `useHome`. Recent activity is the finished sessions from the same response.

**Mock exam placeholder**: `mock-exam-api.ts` returns a static highlight with a status (`coming_soon`, `available`, `in_progress`, `completed`) and a `route` that is null for now. The card is disabled while the route is null. When mock exams are built, change only those function bodies and the Hub and Profile need no edit.

**Fresh numbers**: query keys start with `student`, so login and logout clear them. Finishing a session or changing a bookmark invalidates Home and Stats, and Hub and Stats also refetch when the tab comes back into view.

**Kept the same on purpose**: the Tests tab behaviour (it now shares the start logic through `useQuickSession`), the practice screens, and the theme and layout components.

## Decisions made

**XP and "Top %" are hidden, not faked.** The same for the syllabus "mastered" bar, the commitment picker, the Copilot card, the "On track" chips and the notification bell. None has a data source yet. They can come back with a real source.

**Latest mock score shows a dash.** It reads `getLatestMockScore`, which returns null until mock exams exist.

**Study context removed.** It only held mock data, so it was deleted and the sign-in screens no longer write to it. The subjects count now reads the onboarding data.

## Notes

- `src/services/auth-service.ts` has an uncommitted `logout` change in the working tree that is not part of this work. `profile.tsx` calls it, and it deletes the saved login first and revokes the session on the server in the background.
- Not run on a device or emulator. Nothing was looked at visually, including empty states and the pull to refresh.

## Next steps

- Run the app against the backend and check the three tabs with a new student and with one who has practice history.
- Build real mock exams, then replace the bodies in `mock-exam-api.ts`.

## Test plan

**Automated tests (none added in this repo):**
- [ ] The mobile repo has no test files for these screens - not added in this PR. The backend routes are covered in `PR-backend.md`.
- [x] `npx tsc --noEmit`: no errors.
- [x] `npx eslint src`: no errors.

**Manual checks (local):**
- [ ] Hub, Stats and Profile on the emulator against the backend - not run. Only type check and lint cover the screens.
- [ ] Finish a practice session and see Hub, Stats and Profile update - not run, same reason.
- [ ] Log out and in as another account and see no old data - not run, same reason.
