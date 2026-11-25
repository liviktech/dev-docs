# 2026-11-19 - practice-tests (backend)

## Summary

Students had no way to get questions from the backend. Questions, explanations, bookmarks and practice session tables already existed in the database, but only the admin panel could read questions, and those replies show the correct answer. This PR adds a student `practice` module so the mobile app can start a practice session (by subject, topics, difficulty and length), answer one question at a time with instant feedback, finish and score a session, bookmark questions, and retry incorrect ones.

The new tests pass against the test database, and the app was used against this backend (see the test plan). The correct answer is never sent with a question, and a test checks that.

**Mock exams, timers and mixed-subject sessions are not included on purpose** - see "Next steps" below. This was a choice, not a miss.

## What changed

| File | Change |
|---|---|
| `src/database/migrations/0010_practice-sessions.sql` (new) | Makes `assessment_attempt_modules.duration_seconds` optional (practice has no timer) and adds an index on `assessment_attempts (user_id, assessment_type, status, updated_at)` for the Tests tab. Has a down section. |
| `src/database/seeds/0002_exam-versions.sql` (new) | Adds one active `exam_versions` row for SAT and one for ACT. Safe to run twice. Down step does nothing, same as the first seed. |
| `src/modules/practice/practice.module.ts` (new) | Module wiring. Uses `AuthModule` and `StudentRoleGuard`. |
| `src/modules/practice/practice.controller.ts` (new) | The 8 routes, Swagger docs, `StudentRoleGuard`. Thin: one service call each. |
| `src/modules/practice/practice.service.ts` (new) | Rules: pick questions, create the session, answer, submit and score, summary, bookmarks. Also the pure helpers `buildCatalog` and `buildResult`. |
| `src/modules/practice/repositories/practice.repository.ts` (new) | All SQL for the module. |
| `src/modules/practice/schemas/practice-request.schemas.ts` (new) | Zod request schemas (strict objects, `count` 1 to 20, difficulty 1 to 3). |
| `src/modules/practice/schemas/practice-response.schemas.ts` (new) | Response shapes for Swagger and the DTO types. |
| `src/modules/practice/dto/practice.dto.ts` (new) | Request types inferred from the schemas. |
| `src/modules/practice/interfaces/practice.interfaces.ts` (new) | Row and domain types. |
| `src/app.module.ts` | Registers `PracticeModule`. |
| `src/shared/constants/http.constants.ts` | New error codes: `PRACTICE_NO_QUESTIONS`, `PRACTICE_NOT_AVAILABLE`, `INVALID_OPTION`, `ALREADY_ANSWERED`, `SESSION_SUBMITTED`. |
| `test/integration/practice.int-spec.ts` (new) | 21 integration tests. |
| `test/unit/practice.spec.ts` (new) | 5 unit tests for `buildCatalog` and `buildResult`. |
| `architecture.md` | New section 10e, the status table row, the module tree. The old line saying mobile endpoints "must shuffle options" is changed to match what was built (see "Decisions made"). |

## Why

The mobile Tests tab only had a fixed 5 question demo. The task was to make practice mode work end to end: choose subject, topic or difficulty and length, see the explanation after each answer, move next and previous, bookmark questions, and retry wrong ones. Most of the work was the rules around the existing tables, not new tables. Practice sessions, answers and bookmarks were already modelled in migration 0001 and had never been used.

## How

**No new tables**: a session is an `assessment_attempts` row of type `practice_session` with the chosen filters in `configuration`. It has one module row and one item row per question. Each item points to the exact published question revision that was delivered, so later edits do not change old results. Bookmarks use `question_bookmarks`.

**Routes** (all under `/api/v1`, student token required):

| Route | What it does |
|---|---|
| `GET /student/practice/catalog` | The student's chosen subjects with topics and published question counts per difficulty. Empty ones are left out. |
| `POST /student/practice/sessions` | Starts a session. `mode` is `custom`, `bookmarked` or `incorrect`. Returns fewer questions when fewer exist, 422 `PRACTICE_NO_QUESTIONS` when none. |
| `GET /student/practice/sessions/:sessionId` | Resume, go back, review. Has `result` once finished. |
| `PUT /student/practice/sessions/:sessionId/items/:itemId/answer` | Saves the answer, returns `isCorrect`, `correctOptionId` and `explanation`. |
| `POST /student/practice/sessions/:sessionId/submit` | Scores the session. Safe to repeat. |
| `GET /student/practice/summary` | Bookmarked count, incorrect count, unfinished session, last 5 finished. |
| `PUT` and `DELETE /student/bookmarks/:questionId` | Add or remove a bookmark. Safe to repeat. |

**The answer is hidden until the student answers**: a question has `answer: null` until then, and options never carry `isCorrect`. After answering, `answer` holds the chosen option, whether it was right, the right option and the explanation. The same data comes back when an answered question is loaded again.

**Answers are final**: the save is a single `UPDATE ... WHERE selected_option_id IS NULL`, so a double tap saves only the first answer. A second answer is 409 `ALREADY_ANSWERED`, answering a finished session is 409 `SESSION_SUBMITTED`, and an option from another question is 422 `INVALID_OPTION`.

**Picking questions**: published revision, question not archived, primary topic under the chosen subject, and the chosen topics and difficulty. Questions the student has not answered before come first, then random order. `bookmarked` uses the student's bookmarks. `incorrect` uses questions whose latest answer was wrong, so a question leaves that list once it is answered correctly.

**Exam on every session**: the exam is added to the session `config` when it is read (from the session's exam version), so the app can show "SAT" or "ACT". It is not stored in the saved JSON, so older sessions get it too.

**Other students' sessions are 404**, not 403.

**Kept the same on purpose**: the admin question routes and their replies are unchanged. Mobile never uses them.

## Decisions made

**One subject per session.** Topics are optional and can be several, difficulty is easy, medium, hard or mixed. Mixed-subject sessions were not picked because one session module holds one subject. For bookmarked and retry sessions the module's subject is the subject of the first question.

**One exam per session.** Bookmarks or mistakes can span SAT and ACT, but a session has one exam version. The exam of the first picked question is used and the others are skipped.

**Options keep the admin's order, no shuffle.** `architecture.md` used to say mobile endpoints must shuffle options on the server. A random shuffle per request would change the order when a student goes Previous, so a stable shuffle per item would be needed. That was not done, and the line in `architecture.md` was changed to say so. Needs a decision before launch.

**Only the student's chosen subjects can be practised** (from onboarding). Anything else is 422 `INVALID_SUBJECT`.

**`duration_seconds` made optional instead of storing a dummy value.** A fake 1 second would be wrong data for mock exams later.

**No special rate limit on the new routes.** They use the global limit like the onboarding routes. A session start picks up to 200 candidates, which is cheap, but this could be revisited.

## Notes

- Migration `0010` and seed `0002_exam-versions.sql` must be applied in every environment. Without the exam version rows, starting a session fails with 503 `PRACTICE_NOT_AVAILABLE`. Questions alone do not need them, which is why question upload worked without them. They were applied to the dev database by the user, not by me.
- `npm run format:check` reports three files. One is mine and is fixed. The other two, `src/modules/student/student-onboarding.service.ts` and `test/integration/student-onboarding.int-spec.ts`, were already unformatted and are not touched here.
- The migration down section was written but not run (`migrate:down` was not tried).

## Next steps

- Timed tests, mock exams and mixed-subject sessions. Practice has no timer. Mock exams are a separate product with fixed question sets, modules and scoring. A session with several subjects needs one section per subject in the data.
- Markdown, formula and image rendering. Questions are stored as markdown and the API sends the text as is.
- Changing answers, hints and per-question time tracking. Answers are final, there are no hints, and `time_spent_seconds` stays empty.
- Stable per-item shuffle of options (see "Decisions made").
- Editing questions and admin panel changes. This work only reads existing questions.

## Test plan

**Automated tests (all passing):**
- [x] `test/unit/practice.spec.ts`: `buildCatalog` groups counts by subject, topic and difficulty; `buildResult` scores, counts skipped questions and groups by topic, with "Other" for questions without a topic.
- [x] `test/integration/practice.int-spec.ts`: catalog counts and archived questions left out; filters by topic and difficulty; the session reply has no `isCorrect` and no explanation; fewer questions than asked; 422 for no match, wrong subject and topics of another subject; 400 for a bad body; 503 with no active exam version; right and wrong answer feedback; second answer is 409 and the first stays; option of another question is 422; resume shows answered questions only; submit scores and counts skipped, is safe to repeat, and blocks later answers; bookmarks add, remove and repeat; bookmarked session holds only bookmarked questions; retry incorrect uses the latest answer and shrinks when fixed; summary shows the unfinished session and the exam and subject; another student gets 404; no token gets 401.
- [x] `npm run typecheck` and `npm run lint`: clean.
- [x] `npm test`: 57 passing (unit).
- [x] `npm run test:integration`: 158 passing (full suite, run before the last small change that added `exam` to the session config). After that change only `practice.int-spec.ts` was re-run, 21 passing.

**Manual checks (local):**
- [x] The mobile app was run against this backend by the user. A screenshot of the Tests tab showed a recent session (0/5), "Retry incorrect" with 5 to try again, and the bookmarked card disabled at zero, which matches the data.
- [ ] The full manual checklist in `PLAN.md` section 8 was not run by me (no emulator session on my side).
- [ ] Swagger docs for the new routes were not opened and checked in a browser.
- [ ] `migrate:down` then `migrate:up` for `0010` was not run.
