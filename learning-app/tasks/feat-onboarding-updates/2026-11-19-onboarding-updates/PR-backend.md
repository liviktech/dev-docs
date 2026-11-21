# 2026-11-19 - onboarding-updates (backend)

## Summary

The mobile app had a full six step onboarding flow, but the backend had no onboarding routes and no tables for it. The app only worked through a dev-only mock, so nothing was saved and subjects were hardcoded in the app. This PR adds the student onboarding API and the database tables behind it, so every step is saved, can be resumed after logout, and subjects come from the database.

The new integration tests ran against a local test database and pass. The migrations were not run against the shared dev database by me; the user said they applied them there. The new routes were not tested from a real device.

This PR also fixes ten older integration test failures found while testing: nine are wrong test setup or test assertions, and one is a real bug in the email code resend cooldown. After the fixes the full integration suite passes (137 of 137).

**"Video lessons" and "AP Exams" are not in the backend on purpose** - see "Decisions made" below. This was a choice, not a miss.

## What changed

| File | Change |
|---|---|
| `src/database/migrations/0008_exam-score-range.sql` (new) | Adds `score_min`, `score_max`, `score_step` to `exams` (SAT 400/1600/10, ACT 1/36/1, other rows 1/100/1), then makes them NOT NULL with a check. Down drops them. |
| `src/database/migrations/0009_student-onboarding.sql` (new) | Creates `student_profiles`, `student_subjects`, `student_exam_goals`, `student_learning_goals`, with check constraints. Down drops all four. |
| `src/database/seeds/0001_exams-and-taxonomy.sql` | The exams insert now includes the score range columns, so the seed still works with the new NOT NULL columns. |
| `src/modules/student/student.module.ts` (new) | Module wiring. Imports `AuthModule`. |
| `src/modules/student/student.controller.ts` (new) | Routes under `/v1/student`: `GET onboarding`, `GET profile`, `PUT onboarding/subjects`, `academic`, `goals`, `skill-levels`, `study-preferences`, and `POST onboarding/complete`. Every route returns the full state. |
| `src/modules/student/guards/student-role.guard.ts` (new) | Runs `AccessTokenGuard`, then allows only the `student` role (403 otherwise). |
| `src/modules/student/student-onboarding.service.ts` (new) | Step rules, validation and the step progress logic. |
| `src/modules/student/repositories/student-onboarding.repository.ts` (new) | All SQL for the module. |
| `src/modules/student/schemas/`, `dto/`, `interfaces/` (new) | Zod request and response schemas, DTOs and types. |
| `src/shared/constants/http.constants.ts` | Four new error codes: `INVALID_SUBJECT`, `INVALID_ONBOARDING_DATA`, `ONBOARDING_STEP_NOT_REACHED`, `ONBOARDING_INCOMPLETE`. |
| `src/app.module.ts` | Imports `StudentModule`. |
| `test/integration/student-onboarding.int-spec.ts` (new) | 32 integration tests for the whole flow. |
| `test/integration/question-upload.int-spec.ts` | Its exams insert now includes the score columns. No other change. |
| `src/modules/auth/repositories/email-otps.repository.ts` | New `releaseUnsent`: cancels a code whose email never went out and moves `last_sent_at` back, so the resend cooldown is really freed. `invalidate` is unchanged (still used for expired codes). |
| `src/modules/auth/auth.service.ts` | When sending the code email fails, it now calls `releaseUnsent` instead of `invalidate`. |
| `test/integration/auth-audience.int-spec.ts` | The mobile signup helper now sends `phone`, which signup requires, and the role query sorts by `role::text`. Test only. |
| `test/integration/email-otp.int-spec.ts` | The exam preference query sorts by `exam::text`. Test only. |
| `architecture.md` | Status table, module tree, route table and a new onboarding section. The old "planned onboarding" notes are removed. |

## Why

The user asked for onboarding to work end to end: mobile app, backend and database. Subjects must come from the database through a proper API, "Video lessons" and the AP Exams goal must go, and the plan must be in plain English. The design is in `PLAN.md` next to this file.

The easy part was the routes. The real work was step progress (resume, no skipping ahead) and the validation rules that tie the steps together.

## How

**Four tables**: `student_profiles` has one row per student (current step, academic details, preparation goal, study preferences). `student_subjects` holds each chosen subject with its skill level. `student_exam_goals` holds a target score and exam date per exam. `student_learning_goals` holds the final learning goals. Subjects are saved by `taxonomy_nodes` id, because subject codes repeat across SAT and ACT.

**Subjects from the database**: `GET /student/onboarding` returns the exams offered to the student, each with its active top level subjects, in `position` order. The exams come from `users.target_test`: `sat` gives SAT, `act` gives ACT, and `both`, `other` or no value give both.

**Step progress**: saving step N sets `current_step` to the larger of the current value and N+1, up to 6. Saving a step beyond `current_step` is rejected with 409. `POST onboarding/complete` needs step 6 reached and at least one subject, then sets `users.onboarded_at`. A repeat call keeps the first completion time.

**Validation**: age 5 to 100; graduation year from last year to 7 years ahead; exam date from today to 3 years ahead; target score inside the exam range (stored in `exams`); goals only for exams that have a selected subject; skill levels only for selected subjects.

**Changing subjects later**: replacing the subject list also removes exam goals for any exam left with no subject.

**Kept the same on purpose**: `users.onboarded_at` is the existing completion flag. The plan called it `onboarding_completed_at`, which does not exist, so the real name is used.

## Decisions made

**Four tables, not more.** A larger split was considered. Four keeps one row per student for the profile and still gives each list its own table with a real key.

**"Other" is still accepted at signup.** The backend and database still allow `target_test = other`. Only the mobile app hides it, with a flag. This means other exams can be turned on later without a backend change.

**"Video lessons" and `ap` are rejected by the schema.** The learning styles are only `practice_questions`, `written_explanations` and `interactive_learning`, and the exam goal list is only `sat` and `act`. Nothing for them was added, so no cleanup is needed.

**Exam score range lives in the database.** It is a column on `exams`, so the app never hardcodes score limits.

**The cooldown fix changes only the failed-send path.** Making the cooldown check ignore all invalidated codes was the other option. It would also let a user skip the 60 second wait after a code is locked by 5 wrong guesses, so it was not used. `releaseUnsent` frees the cooldown only for a code that was never delivered.

## Bugs found while testing (already there before, fixed here)

**1. Seven `auth-audience` tests failed.** Test setup bug: the signup body had no `phone`, which the API requires (400). Fixed in the test. The app was fine.

**2. One `email-otp` test failed on exam order.** `exam_code` is an enum ordered sat, act, so `ORDER BY exam` put sat first while the test expected act first. Fixed in the test. The app was fine.

**3. One more `auth-audience` test failed on role order.** Same cause as bug 2: `user_role` is an enum ordered student, admin, so `ORDER BY role` put student first. Fixed with `ORDER BY role::text`. The app was fine.

**4. A failed code email did not free the resend cooldown.** After a mail failure (503) the code was cancelled, but the cooldown still counted it. A retry within 60 seconds got 200 and "code sent", yet no email went out and no valid code existed. Fixed in the app (`releaseUnsent`). This only affected users when the mail provider was failing.

## Notes

- Migrations 0008 and 0009 need to run on every environment. The user said they are applied on their dev database. I did not run or check that.
- The Subjects screen is empty if the database has no SAT and ACT subjects. Run `seeds/0001_exams-and-taxonomy.sql`.
- The local test database had an old migration history that blocked the integration run. It was a local database with no users, so the public schema was reset before the run.
- Migrations 0008 and 0009 were checked going down and up cleanly on that local test database only.

## Next steps

- Run the manual checklist in section 9 of `PLAN.md` on a device.
- Run the manual checklist in section 9 of `PLAN.md` on a device.

## Test plan

**Automated tests (new 32 passing; before the fixes below, 128 of 137 integration tests passed):**
- [x] `test/integration/student-onboarding.int-spec.ts`: all 32 cases pass (state, each step, resume, skipping ahead rejected, validation rules, complete, role guard).
- [x] Unit tests: 53 passing.
- [x] Type check and lint: clean.
- [x] Type check, lint and the 53 unit tests pass again after the nine fixes.
- [x] Full integration suite (`npm run test:integration`) after the fixes: 9 files, 137 of 137 tests pass, including the fixed `auth-audience` and `email-otp` cases.

**Manual checks (local):**
- [x] Migrations 0008 and 0009 down then up on the local test database: clean.
- [ ] Migrations on the shared dev database - applied by the user, not checked by me.
- [ ] End to end from a device or emulator - not run.
