# 2026-11-20 - feat-mock-exams (web)

## Summary

The web app had no mock exams (Home showed a "coming soon" card) and no timer on practice. This PR adds a Mock exams area (list, exam page with a countdown, result with a review, retake), a finished-exams list in Stats, a timer in the top bar for exams and timed practice, and a Timed or Untimed choice when setting up practice.

It needs the backend from `PR-learning-app-backend.md`. Checked with unit and component tests, type check and lint; I did not run it in a browser against the real backend.

**The scaled score on the profile is not included on purpose** - see "Notes" below. This was a choice, not a miss.

## What changed

| File | Change |
|---|---|
| `src/lib/mock-exam/api.ts`, `hooks.ts` | Replaced the placeholder with the real requests and hooks. Home's highlight is now built from the exam list |
| `src/lib/mock-exam/format.ts` (new) | Clock text, seconds left, timer colour, minutes in words |
| `src/components/mock-exam/exam-timer.tsx` (new) | The countdown shown in the middle of the top bar |
| `src/components/mock-exam/exam-runner.tsx` (new) | The exam page: one question at a time, no feedback, change answers, mark for review, submit or finish section |
| `src/components/mock-exam/exam-navigator.tsx` (new) | Question rail and strip showing answered and marked |
| `src/components/mock-exam/exam-review-item.tsx` (new) | One question in the review |
| `src/components/stats/mock-exam-history.tsx` (new) | Finished exams, newest first |
| `src/app/(app)/mock-exams/page.tsx`, `[attemptId]/result/page.tsx` (new) | Exam list and result with review |
| `src/app/(focus)/mock-exams/[attemptId]/page.tsx` (new) | Full-screen exam page |
| `src/app/(app)/stats/page.tsx` | "Mock exam history" section |
| `src/app/(app)/practice/setup/page.tsx`, `src/lib/practice/api.ts`, `format.ts` | Timed or Untimed, time limit chips, suggestion of about 1.5 minutes a question; new fields on the session types |
| `src/components/practice/question-runner.tsx` | Shows the timer for timed practice and finishes the session when it reaches zero |
| `src/components/practice/question-footer.tsx` | Optional label for the last button |
| `src/app/api/[...path]/route.ts`, `src/proxy.ts`, `src/components/app/app-shell.tsx`, `src/app/globals.css` | Allow the mock exam routes, protect `/mock-exams`, add a menu item, add a warning colour |
| `src/lib/mock-exam/format.test.ts`, `src/components/mock-exam/exam-timer.test.tsx` (new) | Tests |

## Why

Students take admin-built exams with a countdown, see their score on submit and review every answer afterwards; they can also time their own practice.

## How

**The server owns the clock.** The page turns the server's seconds left into a clock time once (`deadlineAt`) and the timer counts to it, so a sleeping tab stays right. Each saved answer and each return to the tab re-syncs it. At zero the page asks the server to close the section or the exam, and the server decides.

**No answers during an exam.** The exam page never has the right answer. Choosing an option again clears it. The review is requested only after the attempt is finished.

**Section timers.** Only the open section's questions are shown. Finishing a section asks first and is final; a new section starts at question 1 and its own clock.

**Letters follow what is shown.** Options are lettered by position, and nothing is sorted or shuffled on the client.

## Decisions made

**Retake and review live on the result page.** The list shows the latest score and links to its review; Stats lists every finished attempt.

**Mock exams are a menu item, not a Practice tab.** They have their own area so Practice stays as it is.

## Bugs found while testing (already there before, not fixed here)

**1. Many existing files are not Prettier-clean.** `npm run format` rewrites about 40 files that this PR does not need to change.

## Notes

- **Unintended formatting changes.** I ran `prettier --write` over all of `src`, which reformatted about 40 existing files (line wrapping only, no logic change). I tried to restore them from git and the tool blocked it, so they are still changed in the working tree. Before opening the PR, restore the files this PR does not need (list in the chat message), or tell me and I will redo the intended edits by hand.
- The profile's scaled score stays empty: exams give a raw score and a percentage.
- `Date.now()` is used for the countdown on purpose, so it stays right after the laptop sleeps.

## Next steps

- Restore the unintended formatting changes (see Notes).
- Mobile app (`D:\lm`): same screens.
- Try the exam and the timer in a browser against the real backend, including a section timer running out.

## Test plan

**Automated tests (all passing):**
- [x] `src/lib/mock-exam/format.test.ts`: clock text, seconds left, timer colour, suggested minutes, Home highlight choice
- [x] `src/components/mock-exam/exam-timer.test.tsx`: counts down, expires once, resets on a new deadline, names the section
- [x] `npx vitest run`: 29 passing
- [x] `npm run typecheck`: clean
- [x] `eslint` on the new and changed files: clean

**Manual checks (local):**
- [ ] Taking an exam in a browser against the real backend - not done
- [ ] The look of the timer, runner, result and setup screens, on wide and narrow screens - not looked at
