# 2026-11-20 - home-stats-practice-web (web)

## Summary

The web app had sign in and onboarding but nothing after it. A student who finished onboarding could not see the Hub, Stats or Profile, and could not take a practice test. This PR adds all four to the web app, using the same backend endpoints and the same behaviour as the mobile app (`PR-mobile.md`, `PR-backend.md`). The layout follows the Acely site (left sidebar on desktop, bottom tabs on small screens) with the mobile colour theme. Dates and numbers use US formats.


**Mock exams are not included on purpose** - see "Decisions made" below. This was a choice, not a miss.

## What changed

| File | Change |
|---|---|
| `src/app/api/[...path]/route.ts` | The allowlist now includes the student home, stats, practice and bookmark routes. GET and DELETE send no body. DELETE is exported. |
| `src/lib/api/client.ts` | Added `apiDelete`. |
| `src/proxy.ts` | The matcher now covers `/hub`, `/practice`, `/stats` and `/profile`, so they need a login. |
| `src/lib/practice/api.ts`, `format.ts`, `hooks.ts` (new) | Practice types and endpoints, formatting helpers (session title, en-US dates, accuracy), and the TanStack Query hooks: catalog, summary, session, start, answer, submit, bookmark and quick session. |
| `src/lib/practice/format.test.ts` (new) | Unit tests for the formatting helpers. |
| `src/lib/home/api.ts`, `hooks.ts` (new) | `GET /student/home` and `useHome`. |
| `src/lib/stats/api.ts`, `hooks.ts` (new) | `GET /student/stats`, `GET /student/stats/questions` and their hooks. |
| `src/lib/mock-exam/api.ts`, `hooks.ts` (new) | Placeholder for mock exams. Static highlight with no route, and a latest score that is null. |
| `src/components/ui/card.tsx`, `progress-bar.tsx`, `status-message.tsx`, `chip.tsx`, `section.tsx`, `stat-tile.tsx`, `confirm-dialog.tsx` (new) | Shared building blocks. |
| `src/components/app/app-shell.tsx` (new) | Desktop sidebar (Home, Practice, Stats, Profile), mobile top bar and bottom tabs. The nav links sit in a Suspense boundary because they read the URL. |
| `src/components/app/user-menu.tsx` (new) | The user chip at the bottom of the sidebar: avatar, full name, light and dark switch on the same line. Clicking the chip opens a dropdown with Sign out. |
| `src/components/home/focus-card.tsx`, `mock-exam-card.tsx` (new) | Home cards. |
| `src/components/stats/summary-card.tsx`, `subject-performance.tsx`, `test-history.tsx`, `question-lists.tsx` (new) | Stats sections. Question lists load a page at a time. |
| `src/components/profile/study-profile.tsx` (new) | The onboarding answers with an Edit link per section. |
| `src/components/practice/entry-card.tsx`, `session-row.tsx`, `quick-access.tsx`, `option-section.tsx`, `result-summary.tsx`, `review-item.tsx`, `explanation-card.tsx`, `passage-card.tsx` (new) | Practice entry, setup and result pieces. |
| `src/components/practice/question-runner.tsx`, `question-sidebar.tsx`, `question-topbar.tsx`, `question-footer.tsx`, `mark-for-review.tsx`, `answer-option.tsx` (+ `answer-option.test.tsx`) (new) | The test screen (see "How"). |
| `src/app/(app)/layout.tsx` (new) | Wraps the signed in pages in `AppShell`. |
| `src/app/(app)/hub/page.tsx`, `stats/page.tsx`, `profile/page.tsx` | Home, Stats and Profile on real data, with loading and error states. |
| `src/app/(app)/practice/page.tsx`, `practice/setup/page.tsx` (new) | Practice home (resume, quick start, Bookmarked, Incorrect, recent) and the custom setup page. |
| `src/app/(app)/practice/[sessionId]/result/page.tsx`, `loading.tsx` (new) | Result with score, topic breakdown, Retry incorrect and answer review. |
| `src/app/(focus)/layout.tsx`, `practice/[sessionId]/page.tsx`, `loading.tsx` (new) | The full screen test page. A finished session redirects to its result. |
| `src/app/globals.css` | Added the answer shake and pop animations, turned off when the user prefers reduced motion. |

## Why

The mobile app already had a dynamic Hub, Stats, Profile and practice tests. The web app needed the same so a student can use either one. The backend work was already done for mobile, so this PR is the web client and its proxy rules. The real work was the test screen and keeping the web behaviour the same as mobile.

## How

**Same data path as before**: the browser calls `/api/*`. The route handler keeps the login tokens in httpOnly cookies and forwards to the backend. Only routes on the allowlist pass through. Query keys start with `student`, so login and logout clear them.

**Practice flow matches mobile**: the runner opens at the first unanswered question. A bookmark changes on screen first and rolls back if the call fails. An answer updates the cached session in place. A 409 reply reloads the session so the page shows the truth. Finishing with open questions asks first. Retry incorrect and answer review work like mobile.

**Test screen follows Acely**: a left rail with the logo, the exam name (SAT or ACT) and a numbered list of questions, where bookmarked ones show an icon and answered ones show green or red. A top bar shows the subject and the category of the current question. The passage sits on the left and the question on the right, each scrolling on its own. A bottom bar shows "Question n of N" in the centre with Back and Next in the right corner. "Mark for Review" is the existing bookmark. On small screens the rail becomes a row of numbers. The top bar has an empty centre slot so a timer can be added later.

**Web extras**: keys A to E choose an answer, the arrow keys move between questions, and correct and wrong answers show an icon as well as a colour.

**Mock exam placeholder**: the same as mobile. Only the function bodies in `src/lib/mock-exam/api.ts` change when the real API exists.

**Kept the same on purpose**: the backend, the API shapes, the sign in and onboarding pages, and the theme tokens copied from mobile.

## Decisions made

**Sidebar with Home, Practice, Stats and Profile.** This follows the Acely layout and the four mobile tabs. More items can be added later.

**No mock exam, timer or "Eliminate Answers" yet.** Mock exams do not exist in the backend. The timer and the elimination toggle are not in mobile either, so they were left out. The timer has a slot in the top bar.

**Mono font kept on number figures.** Scores, streak and accuracy use the mono font from the mobile theme. The passage and the question meta line use the normal Inter font.

**Text and layout copy Acely only in structure.** Colours, logo and wording are Prepify's own.

## Notes

- Not checked in a live browser. The checks are type, lint, build and unit tests. The Hub, Stats, Profile, setup, test and result pages, and the dropdown on the user chip, have not been looked at running.
- Next.js 16 with `cacheComponents` needs client hooks that read the URL to sit inside a Suspense boundary. This is why `NavLinks` is separate in `app-shell.tsx`.
- The backend has two uncommitted changes that are not part of this work (see `PR-backend.md`).

## Next steps

- Log in with a real student and check all four areas on desktop and on a phone width.
- Add the timer, and the "Eliminate Answers" toggle if wanted, to the test screen.
- Build real mock exams, then replace the bodies in `src/lib/mock-exam/api.ts`.

## Test plan

**Automated tests (15 passing):**
- [x] `src/lib/practice/format.test.ts`: session titles, counts, en-US dates and accuracy.
- [x] `src/components/practice/answer-option.test.tsx`: idle, selected, correct and wrong states and their labels.
- [x] `npx tsc --noEmit`: no errors.
- [x] `npx eslint src`: no errors.
- [x] `next build`: passes.

**Manual checks (local):**
- [ ] Hub, Stats, Profile, Practice, setup, test screen and result in a browser against the backend - not run, login needs an emailed code. Type, lint, build and tests cover the code only.
- [ ] Finish a practice session and see Hub and Stats update - not run, same reason.
- [ ] Log out from the sidebar chip and log in as another student, and see no old data - not run, same reason.
