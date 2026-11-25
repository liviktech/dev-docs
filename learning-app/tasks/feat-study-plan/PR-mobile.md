# 2026-11-21 - study-plan-and-loader (mobile)

## Summary

The mobile app had a "Cards" tab that only said coming soon, and every loading state was a plain spinner. This PR replaces that tab with a Plan tab backed by the new `/student/plan` routes in `PR-backend.md`, adds a "Today's Plan" block to the Hub, and adds a screen to add your own session. It also adds the shared Prepify loading animation with the caption "Getting ready to learn...", a "Recent exams" list on the Tests tab, and a fix for the mock exam question number bar.

`npx tsc --noEmit` is clean. I did not get the app to show on the Android emulator: after the rebuild it starts and its JavaScript runs with no error, but the emulator window stays black. See "Notes". Nothing here was checked visually.

## What changed

| File | Change |
|---|---|
| `src/services/study-plan-api.ts` (new), `src/hooks/use-study-plan.ts` (new) | Calls and hooks for the plan routes (get, generate, add, change, delete). |
| `src/hooks/use-start-plan-item.ts` (new) | Opens what an item is for: a practice session, a retry session or the mock exams. |
| `src/utils/plan-format.ts` (new) | Day labels, grouping by day, routine summary, questions for a number of minutes. |
| `src/app/(tabs)/plan.tsx` (new) | The Plan tab: routine, days with their sessions, build or refresh the plan, link to add your own and to edit the routine. Pull to refresh and refetch on focus. |
| `src/app/add-plan-item.tsx` (new) | Screen to add a custom session. |
| `src/components/plan/plan-day-section.tsx`, `plan-item-row.tsx`, `today-plan-card.tsx` (new) | Day group, one session row, and the Hub block. |
| `src/components/ui/text-button.tsx` (new) | Small text button used for "Edit study routine". |
| `src/app/(tabs)/cards.tsx` (deleted) | The coming soon placeholder. |
| `src/components/app-tabs.tsx`, `src/components/app-tabs.web.tsx` | The Cards tab is replaced by Plan (calendar icon). |
| `src/app/_layout.tsx` | Registers the `add-plan-item` screen (slides up). |
| `src/app/(tabs)/hub.tsx` | Adds `<TodayPlanCard />` above Quick Access. |
| `src/app/(tabs)/tests.tsx` | Adds a "Recent exams" list (latest 5 mock exams, each opens its result) and refetches it on focus. |
| `src/components/mock-exam/question-palette.tsx` | The number bar keeps the current question centred, and its height is pinned so the dots are never clipped. |
| `src/services/api-client.ts` | Adds `apiPatch`. |
| `assets/lottie/prepify-loading.json` (new) | The loading animation, recoloured to the app palette (blue lines, yellow pencil). |
| `src/components/ui/loading-state.tsx` | Now shows the animation (75px) with the caption. `message` is only the accessible label. |
| `package.json`, `yarn.lock` | Adds `lottie-react-native`. |

## Why

The Cards tab had no content, and a study plan was the next feature students need. The loader was asked to match the web app and use one shared component. The Recent exams list and the number bar were gaps found while testing mock exams.

## How

**Plan data**: the Plan tab and the Hub block read `useStudyPlan`. Marking done or skipped and removing go through the update and delete hooks. The Hub block shows up to three of today's items, an invitation to build a plan when there are none, or the next planned day when today is empty. It shows nothing while loading or on error.

**Starting a session**: practice items start a custom practice session for the subject, topic and a question count that fits the minutes. Retry items use the existing quick session for incorrect questions. Mock exam items open the mock exams list. This reuses `useStartSession` and `useQuickSession`.

**Loader**: `LoadingState` is the one place that shows the animation, so every full screen load gets it without editing the screens. Buttons and small updates keep their small spinners.

**Kept the same on purpose**: auth, the API client behaviour, routing, practice and mock exam logic, and the data fetching of existing screens.

## Decisions made

**Plan replaces Cards in the tab bar.** There was no flashcard feature behind it. The web tab file got the same change so its types still match.

**The JSON sits in `assets/lottie/`, not `src/assets`.** The `@/assets/*` alias points to the root `assets` folder, so putting it under `src` would have clashed with it. `loading-state.tsx` imports it by a relative path.

**The caption is fixed.** `LoadingState` always shows "Getting ready to learn...". The `message` prop is only for screen readers.

## Notes

- `lottie-react-native` is a native module. The dev client has to be rebuilt (`npx expo run:android`). Before the rebuild the app crashed with `IllegalViewOperationException` because the old build had no Lottie view. After the rebuild the log shows `LottieAnimationViewManager` registered.
- The loader size and centring on mobile were not checked on a screen.

## Test plan

**Automated tests (none added in this repo):**
- [ ] The mobile repo has no test files for these screens - not added in this PR. The backend routes are covered in `PR-backend.md`.
- [x] `npx tsc --noEmit`: no errors.
- [x] `npx expo lint`: no errors on the loader change (run earlier in this work). Not re-run after the plan screens were added.

**Manual checks (local):**
- [ ] Plan tab, add screen, Hub block, Recent exams and number bar on the emulator - not possible, the emulator window stayed black (see Notes). Type check covers the code only.
- [ ] Loader size and centring - not possible, same reason.
- [x] Native Lottie module after `npx expo run:android`: `LottieAnimationViewManager` is registered in the log, and the app starts without a crash.
