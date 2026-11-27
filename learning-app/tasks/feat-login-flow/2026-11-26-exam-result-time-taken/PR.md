# 2026-11-26 - exam-result-time-taken

**Companion PR**: `learning-app-backend` - `<backend PR link>` (adds `timeTakenSeconds` to the mock exam attempt response). Merge the backend first. If the app ships first nothing breaks, the time taken line just stays hidden until the backend is live.

## Summary
After finishing a mock exam the result screen showed the score but not how long the exam took, a selected answer option was only marked by its border, the back arrow on the Exams list went back to whatever screen came before (often the last result), and the profile header looked busy. This PR adds the time taken to the score card (backend now returns it), fills the selected option with a soft background, makes Exams back always land on the Tests tab, and tidies the profile header using the existing colours. The time taken was checked live on an emulator ("1m 28s" for a real attempt).

**The analytics scheduler rework is not included on purpose** - see "Next steps" below. This was a choice, not a miss.

## What changed

### learning-app
| File | Change |
|---|---|
| `src/services/mock-exam-api.ts` | Added optional `timeTakenSeconds` to the attempt type. |
| `src/components/practice/result-summary.tsx` | New optional `timeTaken` prop. When given, the score card shows a "Time taken" section under the answered/skipped line. |
| `src/app/mock-exams/result/[attemptId].tsx` | Formats the seconds (for example `1h 2m 5s`, `1m 28s`, `9s`) and passes them to the score card. |
| `src/components/practice/answer-option.tsx` | Selected option gets the `accentSoft` background, and the letter badge turns solid accent with a white letter so it stays readable. |
| `src/app/mock-exams/index.tsx` | Back arrow and Android hardware back now go to `/tests` instead of `router.back()`. |
| `src/app/(tabs)/profile.tsx` | Header redesign: single avatar circle with a soft border, exam name as a pill, stats show value first then label in the regular font, "Practice" renamed to "Last practice". |
| `__tests__/screens/exams/mock-exam-screens.test.tsx` | New test that the result screen shows "Time taken" and `1h 2m 5s`. |

### learning-app-backend
| File | Change |
|---|---|
| `src/modules/mock-exams/mock-exams.service.ts` | Finished attempts return `timeTakenSeconds`: seconds from `started_at` to `submitted_at`, or to `expires_at` for an expired attempt. Null while in progress. |
| `src/modules/mock-exams/schemas/mock-exam-response.schemas.ts` | Added `timeTakenSeconds` (nullable int) to the attempt response schema. |

## Why
Students had no way to see how long they spent on an exam, and the selected answer was hard to spot with only a border change. The Exams back arrow sent students back into a finished result instead of the Tests tab, which felt like a loop.

## How
**Time taken from the server**: the attempt row already stores `started_at` and `submitted_at`, so the backend works the duration out and the app only formats it. The app has no reliable start time of its own (an exam can be resumed on another device).
**Shared score card**: the time is an optional prop on the existing `ResultSummary`, so practice results look exactly as before.
**Back to Tests**: `router.navigate('/tests')` returns to the existing tab screen rather than pushing a new one, and a `BackHandler` listener (only while the Exams screen is focused) does the same for the Android back button.

## Decisions made
**Duration on the backend, not in the app.** Rejected sending raw timestamps and doing the subtraction on the phone, because the phone clock can be wrong and the server already owns the exam clock.
**Expired attempts count up to the deadline.** An expired attempt has no real submit moment, so the deadline is the honest end time.
**Optional field in the app type.** Keeps the app working against an older backend during rollout.

## Bugs found while testing (already there before, not fixed here)
**1. Analytics timer does not fit Vercel.** The analytics aggregation runs on an in-process `setInterval` every 60 seconds. On Vercel serverless the function sleeps between requests, so in production the summaries are likely updated irregularly. Locally it logs "Analytics cycle finished" every minute even when there is nothing new.

## Notes
- If the time taken does not appear on a device, reload the JS bundle (press `r` in Metro). An installed EAS build needs a rebuild to pick up the app change.
- The backend type-aware lint (`tsgolint.exe`) failed to start on this Windows machine, so backend lint was not run.

## Next steps
- Rework analytics aggregation: skip idle runs, trigger after new activity with a debounce, Vercel Cron as fallback, quieter logs. Plan is written up; waiting on the Vercel plan tier (Hobby allows cron once a day) and approval to add `@vercel/functions`.
- Add a test for the Exams back button going to `/tests`.

## Test plan
**Automated tests (40 screen tests, 208 backend unit tests):**
- [x] `__tests__/screens/exams/mock-exam-screens.test.tsx`: result screen shows "Time taken" and `1h 2m 5s`, passes
- [x] `__tests__/screens` (all): 40 passed
- [x] Backend `test/unit`: 208 passed
- [x] App `tsc --noEmit` and backend `npm run typecheck`: clean
- [ ] Backend `npm run lint`: not run, `tsgolint.exe` failed to start on this machine
- [ ] Backend `npm run test:integration`: not run

**Manual checks (local):**
- [x] Finished a mock exam on the Android emulator, result screen showed the completion time ("1m 28s")
- [x] Checked the database: latest attempts have both `started_at` and `submitted_at` set
- [ ] Time taken inside the score card on a device - not yet checked after moving it into the card, covered by the screen test
- [ ] Selected option background, Exams back button and profile header on a device - not yet checked visually
- [ ] Expired attempt showing time up to the deadline - not tested live
