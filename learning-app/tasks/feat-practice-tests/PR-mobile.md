# 2026-11-19 - practice-tests (mobile)

## Summary

The Tests tab only opened one fixed practice test of 5 made-up questions. Nothing was saved, there were no explanations, and a student could not choose what to practise. This PR replaces it with real practice mode: a setup screen (exam, subject, topics, difficulty, number of questions), one question screen with instant feedback and the explanation, Next and Previous, bookmarks, a result screen with answer review and "Retry incorrect", and a Tests tab that shows Continue, Bookmarked, Retry incorrect and recent scores. Everything comes from the backend (`PR-backend.md`).

`npx tsc --noEmit` and `npx expo lint` are clean. The user ran the app on the emulator against the backend and a screenshot of the Tests tab showed recent practice and retry working. I did not run the whole manual checklist myself.

**Stats tab and question formatting are not included on purpose** - see "Next steps" below. This was a choice, not a miss.

## What changed

| File | Change |
|---|---|
| `src/services/practice-api.ts` (new) | Types and one function per backend route. |
| `src/hooks/use-practice.ts` (new) | React Query hooks: catalog, summary, session, start session, answer, submit, bookmark toggle. Answers and bookmarks update the cached session in place. |
| `src/services/api-client.ts` | Added `apiDelete` and the `DELETE` method. |
| `src/utils/practice-format.ts` (new) | Difficulty labels, option letters, session title ("SAT · Mathematics · Easy"), count helper, short date. |
| `src/app/(tabs)/tests.tsx` | Rebuilt. Continue card, Start practice, Bookmarked, Retry incorrect, Recent practice with exam, subject and score. Refreshes when the tab comes into view. Loading and error states. |
| `src/app/practice/setup.tsx` (new) | Setup screen: exam (only if the student has both), subject, topics, difficulty, number of questions. Counts adjust to what exists. |
| `src/app/practice/[sessionId].tsx` (new, replaces `[index].tsx`) | Question screen: bookmark, passage, options, instant feedback, explanation, Previous and Next, Finish, leave confirm. Starts at the first unanswered question. Keeps the colour flash, shake and bounce. |
| `src/app/practice/result/[sessionId].tsx` (new) | Score, topic bars, Retry incorrect, Review answers, Back to Tests. |
| `src/app/practice/[index].tsx` (deleted) | Old static question screen. |
| `src/state/practice-context.tsx` (deleted) | In-memory score. Replaced by the API. |
| `src/data/mock-practice.ts` (deleted) | The 5 made-up questions. |
| `src/app/_layout.tsx` | Removed `PracticeProvider`. |
| `src/app/(tabs)/profile.tsx` | Practice score now comes from the last finished session. Logout clears the cached student data so the next account does not see it. |
| `src/components/practice/answer-option.tsx` (new) | One answer row with idle, selected, correct and wrong states and the animations. |
| `src/components/practice/explanation-card.tsx` (new) | Result line and explanation. Shows "No explanation available" when there is none. |
| `src/components/practice/question-header.tsx` (new) | "Question n of N", progress bar, bookmark and close buttons. |
| `src/components/practice/question-nav.tsx` (new) | Previous, Next and Finish. |
| `src/components/practice/passage-card.tsx` (new) | Scrollable reading passage. |
| `src/components/practice/option-section.tsx` (new) | Titled group of chips on the setup screen. |
| `src/components/practice/practice-entry-card.tsx` (new) | Big card on the Tests tab. |
| `src/components/practice/session-row.tsx` (new) | One recent session row. |
| `src/components/practice/result-summary.tsx` (new) | Score and per-topic bars. |
| `src/components/practice/review-item.tsx` (new) | One question in the answer review. |
| `src/components/ui/choice-chip.tsx` (new) | Selectable pill for single or multiple choice. |
| `src/components/ui/icon-button.tsx` (new) | Round symbol button with a 44pt touch target and an accessibility label. |
| `src/components/ui/loading-state.tsx` (new) | Centered spinner with a message. |
| `src/components/ui/status-message.tsx` (new) | Centered empty or error view with an optional action. |
| `src/components/ui/primary-button.tsx` (renamed from `src/components/auth/primary-button.tsx`) | Moved to `ui/`. Added `variant="secondary"` and `loading`. Existing props and look are the same. |
| `src/components/ui/progress-bar.tsx` (renamed from `src/components/dashboard/progress-bar.tsx`) | Moved to `ui/`, no real change. |
| `src/components/ui/chip.tsx` (renamed from `src/components/hub/chip.tsx`) | Moved to `ui/`, no real change. |
| `src/app/(auth)/login.tsx`, `signup.tsx`, `signup-contact.tsx`, `signup-experience.tsx`, `signup-name.tsx`, `signup-test.tsx`, `verify-email.tsx` | Import path change only, no real change. |
| `src/app/(onboarding)/welcome.tsx`, `src/app/(tabs)/hub.tsx` | Import path change only, no real change. |
| `src/components/dashboard/subject-performance.tsx`, `summary-card.tsx`, `src/components/hub/copilot-card.tsx`, `focus-card.tsx`, `module-card.tsx`, `src/components/onboarding/date-field.tsx`, `onboarding-loader.tsx`, `onboarding-shell.tsx` | Import path change only, no real change. |

## Why

The task was to make the Tests tab a real practice mode: practise by subject, topic or difficulty with a chosen number of questions, see the explanation after every answer (the correct option is shown when the answer is wrong), move to the next and previous question, and bookmark questions and retry wrong ones. The old screen already had the look and animations the user liked, so the work was the data flow, the setup screen and the extra screens.

## How

**All data through React Query**: screens only call hooks, hooks call `practice-api.ts`. Starting a session puts it in the cache, so the question screen opens without another request. An answer writes the reply (correct option and explanation) into the cached session. The bookmark star changes at once and goes back if the request fails.

**One question screen with its own index**: Previous and Next change a local index, so moving back is instant and shows an answered question locked, with colours and explanation. It starts at the first unanswered question, so Continue resumes in the right place.

**Answers are final**: after a tap, all options are disabled until the reply arrives. If the server says the question was already answered or the session is finished, the screen reloads the session.

**Skipping is allowed**: Next is always enabled. On the last question the button is Finish, and if some questions are unanswered the student is asked to confirm. Skipped questions show as skipped in the result.

**Setup counts**: the catalog gives question counts per topic and difficulty. Topics or difficulties with no questions are disabled, and the number of questions choices are capped by what exists (5, 10, 15, 20 or all).

**Exam on titles**: every session title starts with the exam, for example "SAT · Reading and Writing · Mixed". Bookmarked and retry sessions show only the exam, because they can mix subjects.

**Shared components in `ui/`**: following the project component rules, shared pieces live in `src/components/ui/`, and feature pieces live in `src/components/practice/`. Screens hold only logic and compose components.

**Kept the same on purpose**: the colour flash, shake and bounce, the A to D option look, and the stack route setup in `src/app/practice/_layout.tsx` (no change).

## Decisions made

**Exam selector only when needed.** SAT and ACT both have a "Mathematics". If the student has subjects from both exams, an Exam row appears first and subjects are shown per exam. With one exam the row is hidden.

**Moved `PrimaryButton`, `ProgressBar` and `Chip` into `ui/`.** The older shared pieces lived in feature folders (`auth`, `dashboard`, `hub`). The user asked for them to be moved so shared pieces are in one place. The other shared-looking pieces (`auth-field`, `choice-option`, `otp-input`, `option-button`, `choice-card`) were left where they are because they belong to the auth and onboarding flows.

**Plain text questions.** Question text is shown as plain text for now. The backend sends the text as stored.

**Symbols for icons.** The bookmark and close buttons use text symbols instead of an icon library, so they look the same on Android and iOS.

**Bookmarked and retry cards start up to 20 questions.** There is no screen for picking questions from the bookmark list.

## Notes

- Needs the backend from `PR-backend.md`, with migration `0010` and the exam version seed applied. If the seed is missing, starting practice shows the server message "Practice is not available for this exam yet".
- The setup screen is empty ("No questions yet") until published questions exist for the student's chosen subjects.
- After logout, cached student data is cleared. Signing in again also clears it (this was already done in `use-after-auth.ts`).

## Next steps

- Stats tab. Its bookmarked and incorrect lists still use mock data from `src/data/mock-dashboard.ts`. The real data now exists, wiring that screen is a separate task.
- Show markdown, formulas and images in questions. Admin questions are markdown, so formulas, tables and images look raw until a renderer is added.
- Timed tests, mock exams and mixed-subject sessions.
- Changing answers, hints and per-question time tracking.
- Move the remaining shared components (`auth-field`, `choice-option`, `otp-input`, `option-button`, `choice-card`) into `ui/` if the team wants one place for everything.
- A screen to pick questions from the bookmark list instead of starting them all.

## Test plan

**Automated tests (no test runner in this app):**
- [x] `npx tsc --noEmit`: no errors.
- [x] `npx expo lint`: no warnings or errors.

**Manual checks (local):**
- [x] Tests tab on the emulator, shown in a screenshot by the user: Start practice card, bookmarked card disabled at zero, "Retry incorrect" with 5 to try again, and a recent session "Reading and Writing · Mixed" with 0/5. This was before the exam was added to the title.
- [ ] Not run by me: the exam in the recent title ("SAT · ..."), the Exam row on setup with both exams, answer feedback, Previous and Next, bookmark toggle, finish with skipped questions, result review, Retry incorrect, resume with Continue, and network off. The full checklist is in `PLAN.md` section 8.
- [ ] Login, signup and onboarding screens use the moved `PrimaryButton`. Type check passes, but they were not opened on a device after the move.
