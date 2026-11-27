# 2026-11-21 - study-plan-and-loader (web)

## Summary

The web app had no study plan, and every loading state was a plain spinner. This PR adds the study plan (a Plan page, a "Today's plan" block on the Hub, and a page to add your own session) on top of the new `/student/plan` routes in `PR-backend.md`. It also adds one shared loading animation with the caption "Getting ready to learn..." for full page loads, and a "Back to home" link on the login and sign-up pages.

`npm run typecheck` and the 38 vitest tests pass. I could not look at any page in a browser (the browser pane was not signed in), so nothing here was checked visually.

## What changed

| File | Change |
|---|---|
| `src/lib/study-plan/api.ts` (new), `hooks.ts` (new) | Calls and React Query hooks for the plan routes. Query key `["student", "plan"]`, so sign in and sign out clear it. The plan is always refetched on open. |
| `src/lib/study-plan/format.ts` (new), `format.test.ts` (new) | Titles, meta text, day labels, grouping by day, date lists, routine summary. 8 test groups covered by `format.test.ts`. |
| `src/app/(app)/plan/page.tsx` (new) | The Plan page: routine, days with their sessions, build or refresh the plan, link to add your own and to edit the routine. |
| `src/app/(app)/plan/add/page.tsx` (new) | Form to add a custom session. |
| `src/components/plan/plan-day-section.tsx`, `plan-item-row.tsx`, `today-plan.tsx` (new) | Day group, one session row (done, start, skip, remove), and the Hub block. |
| `src/app/(app)/hub/page.tsx` | Adds `<TodayPlan />` under the focus card. No other change. |
| `src/components/app/app-shell.tsx` | Adds a "Plan" item (calendar icon) to the navigation. |
| `src/app/api/[...path]/route.ts` | Allows `student/plan`, `student/plan/generate` and `student/plan/items[/:id]` through the proxy, and exports `PATCH`. |
| `src/lib/api/client.ts` | Adds the `PATCH` method and `apiPatch`. |
| `src/assets/lottie/prepify-loading.json` (new) | The loading animation, recoloured to the app palette (blue lines, yellow pencil). |
| `src/components/ui/loading-animation.tsx` (new) | Shared animation with the caption, 96px, centred. |
| `src/components/ui/status-message.tsx` | `LoadingState` now renders the animation. A `compact` option gives a small block for part of a page. |
| `src/app/(app)/mock-exams/[attemptId]/result/page.tsx` | The review loader is now `compact`, because it is only part of the page. |
| `src/components/auth/auth-shell.tsx` | Adds a "Back to home" link (to `/`) at the top of the shared header. |
| `package.json`, `package-lock.json` | Adds `lottie-react`. |

## Why

Students asked for a plan built from what they told us in onboarding, and a loader that matched the app. The login and sign-up pages had no way back to the landing page.

## How

**Plan data**: the page, the Hub block and the add page all use the hooks in `src/lib/study-plan/hooks.ts`. Marking done or skipped and removing update the cached plan from the reply, so the list does not reload. Adding a session reloads the list so it lands in the right day.

**Starting a session**: a practice item starts a practice session for its subject, topic and a question count that fits the minutes. A retry item starts a retry of incorrect questions. A mock exam item opens the mock exams page. This reuses the existing practice hooks.

**Hub block**: shows up to three of today's items, an invitation to build a plan when there are none, or the next planned day when today is empty. It shows nothing while loading or on error, so the Hub never gets worse because of the plan.

**Loader**: full page loads use the animation (`LoadingState`). Small updates and buttons keep small spinners. About 14 places call `LoadingState`, and none of them changed. Only the mock exam review uses `compact`.

**Kept the same on purpose**: auth, API handling, routing, practice and mock exam logic, and the data fetching of existing pages.

## Decisions made

**The animation is recoloured in the JSON, not with CSS.** Every stroke is the royal blue and the pencil fill is the warm yellow from the app palette. CSS cannot recolour the inner paths reliably.

**Size is set with inline style (96px) and `[&_svg]:size-full!`.** The first two attempts using only Tailwind size classes still rendered huge, because the animation fills its container and does not read the class. This is the third try. The user did not confirm the final size.

**"Back to home" is in the shared auth shell.** That puts it on login, email verification and every sign-up step with one edit. It always goes to `/`, not to the previous step.

## Notes

- Not checked in a browser. The Plan page, Hub block, loader and back link were only type checked.
- The loader in the reference screenshot (a book icon with "Preparing your questions...") does not exist in this code. Only its centred placement was matched.

## Next steps

- Open the web app signed in and look at the loader size and centring, the Plan page and the Hub block.

## Test plan

**Automated tests (38 passing):**
- [x] `src/lib/study-plan/format.test.ts`: `itemTitle`, `itemMeta`, `questionsFor`, `dayLabel`, `groupByDay`, `datesFrom`, `routineSummary`. Passing.
- [x] `npx vitest run`: 8 files, 38 tests passing.
- [x] `npm run typecheck`: no errors.
- [x] `npx eslint src/components/auth`: no errors. The rest of `src` was not linted in this session.

**Manual checks (local):**
- [ ] Plan page, add page and Hub block against the backend - not run. Type check and the format tests cover the code, nothing covers the screens.
- [ ] Loader size and centring in the browser - not run. The browser pane was not signed in.
- [ ] "Back to home" on login and sign-up - not run, same reason.
