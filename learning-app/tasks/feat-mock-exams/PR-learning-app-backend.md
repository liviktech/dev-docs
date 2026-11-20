# 2026-11-20 - feat-mock-exams (backend)

## Summary

The backend had no way to build or take a timed mock exam, and practice sessions had no timer. This PR adds an admin `exams` module to build and publish exam papers from existing questions, a student `mock-exams` module to take them with a server-owned clock, score them and review them after submit, and an optional timer for practice sessions. Questions and answer options are now shuffled once per attempt and stored.

Checked against the real test database: the full integration suite and the new unit tests pass, and migration `0011` went down and up again cleanly. The admin panel, web app and mobile app are not changed in this PR.

**Scaled SAT/ACT scores and adaptive module routing are not included on purpose** - see "Notes" below. This was a choice, not a miss.

## What changed

| File | Change |
|---|---|
| `src/database/migrations/0011_mock-exams.sql` (new) | Timing and shuffle columns, an open-attempt unique index, two indexes. Down step reverses it |
| `src/shared/assessment/clock.ts` (new) | Remaining time, expiry check, 5 second grace for answers |
| `src/shared/assessment/shuffle.ts` (new) | Fisher-Yates shuffle, passage-aware question shuffle, option shuffle, and the check for options that must keep their order |
| `src/modules/exams/` (new) | Admin module: list, get, create, save, publish, withdraw, archive |
| `src/modules/mock-exams/` (new) | Student module: list, start or resume, answer, section submit, submit, review, history |
| `src/modules/practice/` | `timed` and `durationMinutes` on start; `timed` and `remainingSeconds` on the session and the summary; expiry finishes the session; options and questions shuffled and stored; option letters follow the shown position |
| `src/modules/questions/` | `shuffleOptions` on create, edit, import (optional on a row, auto-detected when left out) and in the responses |
| `src/shared/constants/http.constants.ts`, `src/app.module.ts` | New error codes; the two modules registered |
| `architecture.md` | Section 10g, practice section 10e (shuffle and timer), status table, folder list, endpoint table |
| `test/unit/assessment-shuffle-clock.spec.ts`, `test/unit/mock-exams.spec.ts` (new) | Shuffle, clock, scoring, option order, exam body rules |
| `test/unit/practice.spec.ts` | New fixture field and `orderOptions` cases |
| `test/integration/mock-exams.int-spec.ts` (new) | Admin build and publish, the full student flow, section timers, expiry, shuffling |
| `test/integration/practice.int-spec.ts` | Timed sessions and shuffling cases; the "wrong option" helper picks by text now that options are shuffled |
| `test/integration/question-upload.int-spec.ts` | `shuffleOptions` on create, edit and import |

## Why

Admins need to prepare exam papers and set the time, and students need to take them with a countdown, get a score, and review afterwards. Students also asked for a timer on their own practice. The assessment tables already had most of what was needed, so the schema change is small.

## How

**Exams reuse the assessment tables.** An exam is an `assessment_templates` row with versions; a section is an `assessment_modules` row. Editing a live exam makes a new draft version, so students mid-attempt are not affected. Starting an attempt copies durations and points each item at the exact question revision.

**The server owns the clock.** There is no timer job. Every read and write first settles the attempt: an overdue single-timer exam ends as `expired`; an overdue section closes and the next one starts at the moment the last one ended. Answers are accepted until the deadline plus 5 seconds. Closed sections are refused in the answer `UPDATE` itself.

**No answer leaks.** The attempt payload never selects `isCorrect`, the right option or explanations. Only the review route returns them, and only after submit or expiry. A test scans the payload.

**Shuffle once, store it.** Question order (`position`) and option order (`option_order`) are decided when the attempt or session is created. Resume and review read the stored order. Passage questions stay together. Questions flagged `shuffle_options = false` keep their written order.

**Scoring.** Raw score is the number correct, plus a percentage to one decimal. Unanswered counts as wrong. There is a per-subject breakdown.

## Decisions made

**The admin route is `/admin/mock-exams`, not `/admin/exams`.** `GET /admin/exams` already lists SAT and ACT for the admin panel, so using it would have broken that screen.

**Sections have no custom name.** `assessment_modules` has no name column. A section is shown as its subject plus its module number. Adding a name would need another column; I did not add one.

**Practice expiry uses `submitted`, mock exams use `expired`.** Practice has no separate meaning for expiry, and the Tests tab only lists `submitted` sessions.

**Option letters follow the shown position.** Practice and mock exam payloads send `key` as A, B, C, D by position. The stored `option_key` is no longer shown to students.

**An existing integration test was changed.** The "wrong option" helper in `practice.int-spec.ts` assumed the first non-correct option was always "Option B". With shuffling that is no longer true, so it picks by text.

## Bugs found while testing (already there before, not fixed here)

**1. `test/unit/otp.service.spec.ts` fails.** The case "contains the code, expiry and security notice" fails on the code text `4 8 2 9 1 3`. It failed before any of my changes were made (first run of the unit suite), and it is in the email template, not in exams or practice.

## Notes

- Scaled SAT/ACT scores and adaptive module routing (`assessment_module_routes`) are left out for this version.
- Questions are saved into an exam as their current published revision. Editing a question later does not change an exam until the exam is saved again.
- `npm run lint` and `npm run typecheck` pass with zero warnings.

## Next steps

- Admin panel exams feature (list, builder with question picker, timing switch, "Shuffle answer options" and "Shuffle question order" toggles).
- Web app: header timer, mock exam list, runner, result, History review, timed practice setup.
- Mobile app (`D:\lm`): same screens.
- Read the real imported questions for options that should not be shuffled, before launch.
- Fix the OTP email test.

## Test plan

**Automated tests (all passing except one existing failure):**
- [x] `test/unit/assessment-shuffle-clock.spec.ts`: shuffle keeps every item once, repeatable with a seed, passage blocks stay together, clock and grace period
- [x] `test/unit/mock-exams.spec.ts`: percentage, result totals, option letters by position, order-dependent option detection, exam body rules
- [x] `test/integration/mock-exams.int-spec.ts` (20 cases): create, publish, withdraw and edit a live exam; no answer or explanation before submit; resume returns the same attempt; change answers, score, submit twice; review only after submit and in the order seen; expiry; retake and latest score; other student gets 404; section lock, section expiry, section submit; shuffle is stable, moves the answer between attempts, respects `shuffle_options = false` and `shuffleQuestions: false`
- [x] `test/integration/practice.int-spec.ts`: timed session countdown, validation of the duration, expiry finishes the session, stable option order, answer position varies
- [x] `test/integration/question-upload.int-spec.ts`: `shuffleOptions` default, edit, import auto-detect
- [x] `npm run typecheck` and `npm run lint`: clean
- [x] `npm run test:integration`: 191 passing
- [ ] `npm test`: 92 passing, 1 failing (the OTP email test above, not related to this change)

**Manual checks (local):**
- [x] Migration `0011` down then up on the test database: both completed
- [ ] Not checked by hand: the admin, web and mobile screens do not exist yet, and I did not call the endpoints from Swagger
