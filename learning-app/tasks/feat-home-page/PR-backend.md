# 2026-11-20 - home-and-stats-api (backend)

## Summary

The mobile Hub, Stats and Profile screens ran on made-up data, so a student's real practice never showed up there. This PR adds two read-only student endpoints, `GET /api/v1/student/home` for the Hub and Profile, and `GET /api/v1/student/stats` plus `GET /api/v1/student/stats/questions` for the Stats screen. All numbers come from the practice tables that already exist, so there is no migration.

Backend `npm run typecheck` is clean. `test/unit/home.spec.ts` passes (8 tests) and `test/integration/practice.int-spec.ts` passes (26 tests) against the real test database.


## What changed

| File | Change |
|---|---|
| `src/modules/home/` (new) | `home.module.ts`, `home.controller.ts`, `home.service.ts`, `home.logic.ts`, `repositories/home.repository.ts`, `interfaces/home.interfaces.ts`, `schemas/home-response.schemas.ts`. `GET /student/home` returns the practice summary plus `displayName`, `streakDays` and the primary `exam` (code, name, target score, exam date, days remaining, or null). |
| `src/modules/stats/` (new) | `stats.module.ts`, `stats.controller.ts`, `stats.service.ts`, `repositories/stats.repository.ts`, `schemas/stats-request.schemas.ts`, `schemas/stats-response.schemas.ts`, `dto/stats.dto.ts`, `interfaces/stats.interfaces.ts`. `GET /student/stats` returns totals, per subject accuracy and the last 20 finished sessions. `GET /student/stats/questions?kind=incorrect or bookmarked&page&pageSize` returns a paged review list. |
| `src/modules/practice/practice.service.ts` | New `finishedSessions(userId, limit)`. `summary()` now uses it, so the logic is not copied. No change in what `summary()` returns. |
| `src/modules/practice/practice.module.ts` | Now exports `PracticeService` so Home and Stats can use it. |
| `src/modules/practice/schemas/practice-response.schemas.ts` | `summarySchema` is exported so the home schema can extend it. No real change. |
| `src/app.module.ts` | Registers `HomeModule` and `StatsModule`. |
| `architecture.md` | New section "10f. Student home and stats" with the routes, streak rule, days remaining, incorrect list rule and what is not built yet. Also a status table row and the module tree. |
| `test/unit/home.spec.ts` (new) | 8 tests for `computeStreak` and `daysRemaining`. |
| `test/integration/practice.int-spec.ts` | New `describe('stats')` and `describe('home')`. |

## Why

The Hub, Stats and Profile screens were mock data. Practice sessions, answers and bookmarks were already saved, so most of the work was reading them in the right shape, not adding tables. The Hub needs one fast call on open, which is why Home returns everything in one response.

## How

**One call for the Hub**: `GET /student/home` runs the profile, primary exam, active days and `practice.summary` in `Promise.all` and merges them. The mobile Hub and Profile read from it.

**Streak**: the number of consecutive days, in the student's own timezone (`users.timezone`), that have a submitted practice session. If today has none yet but yesterday does, the streak still counts, so it does not drop to 0 in the morning. The repository looks back 400 days.

**Days remaining**: calendar days from today to `student_exam_goals.exam_date`, in the student's timezone. It is null when there is no exam date.

**Stats totals and subjects**: counted from `assessment_attempt_items` of submitted practice sessions, joined to topic and subject. Only subjects the student has answered in are returned.

**Incorrect list**: uses the latest answer per question (`DISTINCT ON`), so a question answered wrong and later right does not stay in the list. It returns `yourAnswer` and `correctAnswer`. The bookmarked list leaves both null.

**Kept the same on purpose**: every route is student only (`StudentRoleGuard`), mobile route names have no audience keyword, input uses strict Zod schemas, and all SQL is parameterised.

## Decisions made

**XP, Top % and mock exam scores are not returned.** There is no source for them. Faking them in the API would only move the mock data from the app to the server. The mobile app hides XP and Top %, and shows a dash for the latest mock score until mock exams exist.

**The incorrect list can differ from the retry count.** Retry incorrect on the Tests tab uses the current published version of a question. The Stats list shows the version the student actually answered. If an admin edits a question, the two numbers can differ a little. This is written in `architecture.md`.

**Stats reuses `PracticeService`.** Recent finished sessions come from the practice module instead of a second copy of the query, so the two screens always agree.

## Notes

- No database migration is included. Everything reads existing tables (`assessment_attempts`, `assessment_attempt_modules`, `assessment_attempt_items`, `question_bookmarks`, `student_exam_goals`, `user_exam_preferences`, `users`).
- `src/modules/mail/email-templates.ts` has an uncommitted change in the working tree that is not part of this work (it removes the spacing between OTP digits). It makes the existing unit test `test/unit/otp.service.spec.ts > buildVerificationEmail > contains the code, expiry and security notice` fail, because that test expects `4 8 2 9 1 3`. I did not touch it or the test. Decide whether the template or the test is right before merging that change.

## Next steps

- When mock exams exist, add their latest score to the home response so the Profile tile fills in.
- Fix the OTP email template or test mismatch above.

## Test plan

**Automated tests (all passing for this work, one unrelated failure):**
- [x] `test/unit/home.spec.ts`: `computeStreak` (today, yesterday still counts, gap resets, empty) and `daysRemaining` cases. 8 passing.
- [x] `test/integration/practice.int-spec.ts` `stats`: empty state, counts and lists with correct `yourAnswer` and `correctAnswer`, bookmarked answers null, bad `kind` returns 400, no token returns 401.
- [x] `test/integration/practice.int-spec.ts` `home`: display name, streak and summary, no token returns 401. Whole file 26 passing.
- [x] Backend type check (`npm run typecheck`): no errors.
- [ ] `test/unit/otp.service.spec.ts`: 1 failing, caused by the unrelated template change described in Notes.

**Manual checks (local):**
- [ ] Called the new routes by hand from a running server - not done. The integration tests call them over HTTP against the real test database instead.
