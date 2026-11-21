# 2026-11-21 - study-plan (backend)

## Summary

Students had no study plan. Onboarding already collects daily study time, study days, preferred times, skill levels, test date and goals, but nothing used them. This PR adds a `study-plan` module with one table (`study_plan_items`) and five routes under `/student/plan`. The server builds the next 14 days of sessions from the onboarding answers and the student's real results, and the student can also add, move, finish, skip or delete sessions.

Unit tests (30) and the type check pass. The integration tests were written but not run here, because I did not run them against a database. There are no reminders or push notifications, on purpose.

**Reminders and notifications are not included on purpose** - the request was for no notifications right now. This was a choice, not a miss.

## What changed

| File | Change |
|---|---|
| `src/database/migrations/0014_study-plans.sql` (new) | Creates `study_plan_items` with checks on type, status, source, time of day and minutes (5 to 240), and an index on `(user_id, plan_date)`. Down migration drops the table. |
| `src/modules/study-plan/study-plan.logic.ts` (new) | Pure `generatePlan` function and its helpers (`studyDates`, `dayBlocks`, `addDays`, `withoutKept`). No database access. |
| `src/modules/study-plan/study-plan.service.ts` (new) | Loads the student's routine, subjects, topics and accuracy, runs the generator, and handles add, change and delete. |
| `src/modules/study-plan/repositories/study-plan.repository.ts` (new) | All SQL for the module. |
| `src/modules/study-plan/study-plan.controller.ts` (new) | The five routes, behind `StudentRoleGuard`. |
| `src/modules/study-plan/schemas/*`, `dto/study-plan.dto.ts`, `interfaces/study-plan.interfaces.ts` (new) | Zod request and response schemas, DTO and types. |
| `src/modules/study-plan/study-plan.module.ts` (new) | Wires the module. |
| `src/app.module.ts` | Registers `StudyPlanModule`. |
| `src/shared/constants/http.constants.ts` | Adds the `STUDY_PLAN_NO_SUBJECTS` error code. |
| `architecture.md` | Adds the study plan row and a new section 10h that describes the routes and the generator. |
| `test/unit/study-plan.spec.ts` (new) | 30 unit tests for the generator, dates and request schemas. |
| `test/integration/study-plan.int-spec.ts` (new) | Route tests against a real database. |

## Why

The plan is the main thing students asked for after practice and mock exams. The inputs already existed, so the real work was the generator rules and keeping the student's own sessions safe from regeneration.

## How

**One table, no plan header**: the inputs (daily minutes, study days, preferred times) stay on `student_profiles` and are edited in onboarding. `source` is `auto` for generated rows and `custom` for the student's own. Planning again only replaces `auto` rows that are still `pending`. Custom, done and skipped rows are never touched.

**Routes**:
- `GET /student/plan?from&to` returns the items between two `YYYY-MM-DD` dates (at most 62 days, default today plus 13 days) with the routine and the test date.
- `POST /student/plan/generate` builds the next 14 days. It returns 422 `STUDY_PLAN_NO_SUBJECTS` when no chosen subject has questions.
- `POST /student/plan/items` adds a custom item. `practice` needs a `subjectId`. `mock_exam` and `retry_incorrect` ignore subject and topic.
- `PATCH /student/plan/items/:id` marks `done`, `skipped` or `pending`, moves to another `date`, or changes `minutes` or `timeOfDay`.
- `DELETE /student/plan/items/:id` removes an item.

**Generator rules** (`generatePlan`): days follow `studyDays` and stop the day before the test. The daily minutes are split into sessions of about 30 minutes over the preferred times. Subjects share the time by skill level (weaker gets more, and more again with the improve weak subjects goal), adjusted by real accuracy once there are 5 answers. Topics rotate weakest first. Every third study day ends with a retry of incorrect questions. Every seventh study day is a full mock exam when the student wants them or the test is within 30 days. Items that were finished or skipped stay and count toward their day, so planning again does not double a day.

**Kept the same on purpose**: no existing table, route or migration was changed. Another student's items return 404.

## Decisions made

**The generator is a pure function.** It takes plain data and returns planned items, so the rules are tested without a database. The service only gathers the inputs and saves the result.

**No plan header table.** A second place for daily minutes and study days would drift from onboarding, so the routine is read from `student_profiles` every time.

**Generated rows are replaced, custom rows are not.** Planning again is safe to press at any time.

## Notes

- Migration `0014` has to be run (`npm run migrate:up`) before the routes work.
- The new files in `src/modules/study-plan` use LF line endings, the same as the rest of `src`.

## Next steps

- Run `npm run test:integration` against a local database and fix anything it finds.
- Reminders, if they are wanted later.

## Test plan

**Automated tests (30 passing, integration not run):**
- [x] `test/unit/study-plan.spec.ts`: `studyDates`, `dayBlocks`, `generatePlan` (subject weighting, accuracy adjustment, topic rotation, retry every third day, mock exam every seventh day or near the test, no subjects), `addDays`, request schemas and `withoutKept`. 30 passed.
- [x] `npm run typecheck`: no errors.
- [ ] `test/integration/study-plan.int-spec.ts`: written for every route (student only, empty plan, generate and the 422, replace pending but keep done and custom, other students, add, change and delete, foreign subject and topic, bad body, window limits). Not run, because no database was set up in this session.
- [ ] `npm run lint` and `npm run format:check`: not run.

**Manual checks (local):**
- [ ] Migration `0014` up and down against a database - not run. The integration tests cover it once they are run.
- [ ] Routes called by hand - not run. The web and mobile apps call them, see `PR-web.md` and `PR-mobile.md`.
