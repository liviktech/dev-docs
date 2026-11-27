# Plan: Admin-built mock exams and timed practice tests

Status: all four phases are built (backend, admin panel, web, mobile); see the `PR-*.md` files here. Nothing has been tried end to end in a browser or on a phone yet. Web has stray formatting changes, see `PR-learning-web-app.md`.

Changes made while building the backend:

- The admin API is `/api/v1/admin/mock-exams`, not `/admin/exams` (that path already lists SAT and ACT).
- Sections have no custom name. `assessment_modules` has no name column, so a section is shown as its subject plus its module number. The admin form picks a subject for each section.
- `marked_for_review`, `submitted_at` on modules and the score columns were not added: `is_flagged`, `completed_at` and `raw_score` already existed.
- The unique index `one_open_mock_attempt_idx` was added so a double tap on Start cannot create two attempts.
- The student attempt routes are `/student/mock-exams/:examId/attempts` (start) and `/student/mock-exams/attempts/:id/...` (everything else).

## 1. Goal

- Admins build exam papers in the admin panel from the existing question bank (including imported questions) and choose the timing.
- Published exams appear as mock exams in the web app and the mobile app, where students take them.
- Students can also turn a timer on for their own practice tests. Untimed practice stays as it is today.
- The timer is shown in the top header, centred, for both mock exams and timed practice.

## 2. Decisions (confirmed)

| Topic | Decision |
|---|---|
| Answers during a mock exam | Hidden. No instant feedback. Correct answers and explanations are only viewable after submit |
| Review | After submit, the student opens the exam from History and sees every question, their answer, the correct answer and the explanation |
| Retakes | Unlimited. Each retake is a new attempt. The latest attempt's score is the one shown |
| Timing | Single exam timer, or optional section-based timers (admin chooses per exam) |
| Scoring | Raw score (correct / total) plus percentage. No scaled SAT/ACT score in v1 |
| Who configures what | Admins build exams only. Students configure only their own practice tests |
| Shuffling | Answer options are shuffled for every new attempt, in both mock exams and practice tests. Question order is shuffled too (in mock exams the admin can switch it off per exam). See section 5a |

## 3. What already exists (reuse it)

- Tables from migration `0001`: `assessment_templates`, `assessment_template_versions`, `assessment_modules` (has `duration_seconds`, `allow_review`), `assessment_module_questions`, `assessment_attempts`, `assessment_attempt_modules` (has `duration_seconds`, `elapsed_seconds`), `assessment_attempt_items`. `assessment_type` already has `mock_exam`.
- Practice sessions reuse the attempt tables, and `duration_seconds` is already nullable (migration `0010`).
- Web `QuestionTopBar` already has an empty `center` slot for the timer.
- Placeholders to replace: mobile `D:\lm\src\services\mock-exam-api.ts`, web `learning-web-app/src/lib/mock-exam/api`. The Hub card already handles every status.
- Admin panel has `features/content` (questions, import) as the pattern to copy. No exams feature yet.

## 4. Design principles

1. **The server owns the clock.** The client timer is display only. The server stores start time and deadlines and rejects late answers. Closing the app or editing the client timer cannot extend time.
2. **Expiry is handled lazily.** Any read or write on an attempt past its deadline finalises it (`expired`, scored on what was answered). An optional periodic sweep covers attempts nobody reopens.
3. **No answer leaks.** The attempt payload never contains `isCorrect`, the correct option or explanations until the attempt is submitted or expired.
4. **Mock answers are changeable until submit** (different from practice, where answers are final).
5. **One timer component per client**, shared by mock exams and timed practice.
6. **Shuffle once per attempt, on the server, and store it.** The shuffled order is saved when the attempt is created. Resuming, going back and the review page all use the saved order, so nothing moves on refresh.

## 5. Timing model

An exam has a `timing_mode`:

- `overall`: one countdown for the whole exam. Students move freely between all questions.
- `per_section`: each section (module) has its own countdown. Sections run in order. When a section's time ends or the student finishes it, it locks and the next section starts. The student cannot go back to a locked section during the attempt. The timer in the header shows the current section's time, with the section name.

Section data uses the existing `assessment_modules` rows (one per section, each tied to a subject section, with its own `duration_seconds`).

For `overall` mode, an exam may still have several modules purely for grouping questions on screen. The countdown is then the exam total.

## 5a. Shuffling (mock exams and practice tests)

Goal: the same question does not keep the right answer in the same place. If a student sees a question with the correct answer at A, the next time that question appears (a retake, a later practice session) the options are in a new random order, so the answer may be at C.

**Rules**

- **Options**: shuffled for every item in every new attempt or session, for mock exams and practice. The letters A, B, C, D are always assigned by displayed position, never stored with the option.
- **Questions**: the question order is shuffled when the attempt is created.
  - Mock exams: an exam setting `shuffle_questions` (admin toggle, default on). Per-section exams shuffle inside each section only, so questions never move between sections.
  - Practice: always shuffled (it already picks randomly; the order is now also stored).
- **Retakes and repeat questions** get a fresh shuffle each time, because each attempt or session makes its own order.
- **Stable within an attempt**: the order is generated once at creation with an unbiased shuffle (Fisher-Yates), stored, and reused for resume, navigation and review.
- **Never shuffled** (a question stays in its written order):
  - Questions flagged `shuffle_options = false` by the admin. This is for options such as "All of the above", "None of the above", "Both A and B", options that refer to each other by letter, and ordered or numeric choices such as ACT "Point A in Paragraph 1".
  - Questions with fewer than 2 options (nothing to shuffle).
- **Passage groups stay together**: questions that share one reading passage are kept as a block. The blocks are shuffled and, inside a block, the question order is kept (so questions that refer to earlier ones still make sense). Questions outside a passage shuffle freely.
- **Review**: shows the questions and options in the order the student saw them.

**Admin side**

- Question form (and bulk import): new checkbox "Shuffle answer options" (default on). Import file gets an optional `shuffleOptions` column.
- Import default: set to off automatically when any option text matches "all of the above", "none of the above", "both", or refers to another option by letter. The admin can change it.
- Exam form: toggle "Shuffle question order" (default on).

## 6. Phase 1: Backend (`learning-app-backend`)

### 6.1 Migration `0011_mock-exams.sql`

- `assessment_template_versions`: add `timing_mode text NOT NULL DEFAULT 'overall'` (check in `overall`, `per_section`) and `total_duration_seconds integer` (required when `overall`, null when `per_section`; check `> 0`).
- `assessment_attempts`: add `expires_at timestamptz` (set for `overall` attempts and practice timed sessions) and `timing_mode text`.
- `assessment_attempt_modules`: add `started_at timestamptz`, `expires_at timestamptz` (per section), and `submitted_at timestamptz` so locked sections are known.
- `assessment_attempt_items`: add `marked_for_review boolean NOT NULL DEFAULT false`. Mock attempts need this, and it also lets practice mark-for-review be saved if it is not already.
- Indexes: `assessment_templates (assessment_type, exam_version_id) WHERE archived_at IS NULL`, and `assessment_attempts (user_id, assessment_type, template version, created_at DESC)` for "latest attempt".
- Add a `score` jsonb or columns `correct_count`, `total_count`, `percentage` on `assessment_attempts` if the current schema does not already store the result. Check `assessment_attempts` first.
- Shuffling columns:
  - `question_revisions`: add `shuffle_options boolean NOT NULL DEFAULT true`.
  - `assessment_template_versions`: add `shuffle_questions boolean NOT NULL DEFAULT true`.
  - `assessment_attempt_items`: add `option_order uuid[]` (the option ids in the order shown to the student). Question order needs no new column, because the item `position` is assigned from the shuffled order when the items are created.
- Down migration reverses everything.

### 6.2 Admin module `exams` (`/api/v1/admin/exams`, admin guard)

- `POST /admin/exams`: body `{ name, examId, instructions?, timingMode, totalDurationMinutes?, sections: [{ name or taxonomySectionId, durationMinutes?, questionIds[] }] }`
  - `overall`: `totalDurationMinutes` required, sections optional (default one section).
  - `per_section`: every section needs `durationMinutes`.
  - Presets in the UI: 30, 45, 60, 90, 120 minutes, plus custom. The API accepts 1 to 360.
- `GET /admin/exams` (filters: status, exam, search), `GET /admin/exams/:id`
- `PATCH /admin/exams/:id`: editing a published exam creates a new draft version. Students mid-attempt keep the snapshot they started with.
- `POST /admin/exams/:id/publish`, `POST /admin/exams/:id/unpublish`, `DELETE /admin/exams/:id` (archive only)
- Validation: at least one question, no duplicate questions in an exam, only published question revisions, all questions belong to the exam's own exam (SAT or ACT), position order kept.
- Publishing is blocked with a clear error if rules above are broken.

### 6.3 Student module `mock-exams` (`/api/v1/student/mock-exams`)

| Route | Purpose |
|---|---|
| `GET /student/mock-exams` | Published exams for the student's exam. Each has name, duration summary, question count, timing mode, and the student's status (not started, in progress, completed) with the latest score |
| `POST /student/mock-exams/:templateId/attempts` | Start. Snapshots the revisions and durations, sets `started_at` and deadlines. If one is already in progress it returns that one |
| `GET /student/mock-exams/attempts/:id` | Resume. Returns questions (no answers), saved selections, marked flags, `timingMode`, `remainingSeconds` for the active timer, current section |
| `PUT /student/mock-exams/attempts/:id/items/:itemId/answer` | Save or change an answer, optional `markedForReview`. Rejected after the active deadline or if the section is locked |
| `POST /student/mock-exams/attempts/:id/sections/:moduleId/submit` | Per-section mode only: finish a section early and move to the next |
| `POST /student/mock-exams/attempts/:id/submit` | Final submit. Idempotent. Stores raw score and percentage |
| `GET /student/mock-exams/attempts/:id/review` | Only for submitted or expired attempts. Every question, chosen option, correct option, explanation. Otherwise `409 REVIEW_NOT_AVAILABLE` |
| `GET /student/mock-exams/history` | The student's finished attempts, newest first, each with score and percentage. Used by the History view |

- In `per_section` mode, answering an item in a past or future section is rejected.
- Scoring: `correct_count / total_count`, percentage rounded to one decimal, plus a per-subject breakdown. Unanswered counts as wrong.
- The latest attempt per exam is flagged so the Hub and exam list show its score.
- Other students' attempts return `404`.

### 6.4 Practice changes

- `createSessionRequestSchema` gets `timed: boolean` (default false) and `durationMinutes` (5 to 180, required when `timed`).
- Timed sessions set the module `duration_seconds` and the attempt `expires_at`. Session payloads include `timed` and `remainingSeconds`.
- When time runs out the session auto-submits using the same shared expiry logic as mock exams.
- Untimed behaviour is unchanged. Answers in practice stay final with instant feedback.

### 6.4a Shuffling in the backend

- A shared `ShuffleService` (pure functions, easy to unit test, takes an injectable random source):
  - `shuffleQuestions(items, { keepPassageGroups })`: shuffles passage blocks and loose questions, keeps order inside a block.
  - `shuffleOptions(optionIds, { enabled })`: returns the order to store. Returns the admin order untouched when `shuffle_options` is false.
- Used when an attempt is created, by both `mock-exams` and `practice`: assign item `position` from the shuffled question order and write each item's `option_order`.
- Question payloads (`GET` attempt or session, review) return options ordered by the item's `option_order`. The `isCorrect` rule is unchanged, so shuffling never leaks the answer.
- `architecture.md` currently says practice options keep the admin's order with no server shuffle. Replace that with the new behaviour.
- Admin question create, update and import accept `shuffleOptions`. The admin question response includes it.
- Bookmarked and incorrect-retry practice sessions also shuffle, so a retry of a missed question does not show the answer in the same place.

### 6.5 Shared pieces

- A small shared `AttemptClockService`: computes remaining time, applies lazy expiry, handles the grace period (suggest 5 seconds for network delay).
- Update `architecture.md` (new section for exams and mock exams, status table, migration note) in the same change.

### 6.6 Tests

- Unit: clock and expiry, per-section locking, scoring, timing-mode validation.
- Unit (shuffling): every option appears exactly once after shuffling, locked questions keep their order, passage blocks stay together and keep inner order, per-section shuffle never moves a question across sections, a seeded random source gives a repeatable result.
- Integration: create and publish an exam, start, answer, expire, submit twice, review before and after submit (no `isCorrect` or explanation before), retake and latest-score logic, other-student 404, timed practice.
- Integration (shuffling): resume returns the same order as the first read, review order equals the order the student saw, a new attempt of the same exam gets a new order (over many runs the correct option's position is not always the same), `shuffle_options = false` questions are unchanged, no `isCorrect` in payloads.

## 7. Phase 2: Admin panel (`learning-app-admin`)

New `src/features/exams/` with `api`, `queries`, `schemas`, `pages`, `components`, following `features/content`.

- **Exams list page** (`/exams`): name, exam, timing mode, total duration, question count, status (draft or published), actions (edit, publish or unpublish, archive).
- **Create and edit page** (`/exams/new`, `/exams/:id/edit`):
  - Details: name, exam (SAT or ACT), instructions.
  - **Timing mode switch**: "One timer for the whole exam" or "Separate timer per section".
    - Whole exam: duration presets 30, 45, 60, 90, 120 plus custom.
    - Per section: each section row has its own duration with the same presets.
  - **Sections**: add, rename, reorder, remove. A single-section exam is the simple case.
  - **Question picker**: searchable table of the existing question bank (filters: subject, topic, difficulty, imported or created), multi-select into the chosen section, with a selected list that supports reorder and remove, a running count and total time shown.
  - Save as draft, then Publish. Publish shows validation problems inline.
- Add routes in `routes.tsx` and an "Exams" entry in the dashboard sidebar.
- Question form and import page: "Shuffle answer options" checkbox. Exam form: "Shuffle question order" toggle.
- Tests next to the pages, like `content-pages.test.tsx`.

## 8. Phase 3: Web app (`learning-web-app`)

- **Timer component** `ExamTimer`:
  - Shown in `QuestionTopBar`'s `center` slot, for mock exams and timed practice only.
  - Shows `mm:ss` (or `h:mm:ss`), and the section name in per-section mode.
  - Syncs from the server's `remainingSeconds` on load and after each answer save; counts down with a monotonic clock so it does not drift or jump when the tab sleeps.
  - Warning style at 5 minutes and 1 minute left. At zero it submits (or moves to the next section) automatically.
- **Mock exam list** on the Tests area, plus a real `lib/mock-exam/api` replacing the placeholder, so the Hub card shows the next or in-progress exam.
- **Exam runner**: reuse `question-runner`, `question-sidebar` (answered, flagged, current states), `mark-for-review`, with feedback hidden. In per-section mode, only the current section's questions are navigable. A submit dialog lists unanswered and flagged questions, and warns that finishing a section is final in per-section mode.
- **Result page**: raw score, percentage, per-subject breakdown, button to review and to retake.
- **History**: finished mock exams listed (newest first). Opening one shows the full review: each question, the student's answer, the correct answer, the explanation (reuse `review-item`, `explanation-card`, `result-summary`).
- **Practice setup**: add a "Timed / Untimed" toggle and a duration picker (suggest about 1.5 minutes per question). The practice runner shows the timer only when timed.
- A "Resume" prompt for an in-progress exam.
- **Shuffling**: render questions and options exactly in the order the API returns them (do not sort or shuffle on the client). Option letters are labelled by displayed position. Review shows the same order the student saw.

## 9. Phase 4: Mobile app (`D:\lm`, Expo)

- Replace `src/services/mock-exam-api.ts`; add a mock exam hook and a route group (list, runner, result, history and review).
- Header timer centred, same behaviour as web.
- Shuffling: render in the order the API returns, label A, B, C, D by position, never cache or re-sort options locally.
- Practice setup: timed toggle and duration picker.
- App backgrounding: on return to foreground, re-fetch `remainingSeconds` from the server. If the time has expired, show a "time's up" state and the result.
- Before writing code, read the installed Expo SDK version and the matching docs (per `AGENTS.md`). Run `npx expo lint` and `npx tsc --noEmit` before finishing.

## 10. Implementation order

1. Backend (migration, shared clock, admin exams, student mock exams, practice timer, tests, `architecture.md`).
2. Admin panel exams feature.
3. Web app.
4. Mobile app.

Each phase ends with a check: create an exam in admin, publish it, take it on web and mobile, let a timer run out, submit, retake, and open the review from History.

## 11. Out of scope for v1

- Scaled SAT or ACT scores and score conversion tables.
- Adaptive module routing (`assessment_module_routes` exists in the schema but is not used).
- Pausing an exam, or scheduling exams for specific dates or groups.
- Practice test answer behaviour changes (still final with instant feedback).

## 12. Risks to watch

- **Shuffling must be stable**: if the order is regenerated on every read, options would jump around. It is stored once per attempt and covered by a resume test.
- **Shuffling can break a question**: options like "All of the above" or lettered references make no sense when moved. The per-question `shuffle_options` flag and the import auto-detect exist for this. Review real imported questions before launch.

- **Clock correctness** is the riskiest part: test expiry, grace period, double submit and clock drift carefully.
- **Editing a published exam** must never change attempts already started or finished; this relies on snapshotting revisions and durations at start.
- **Answer leaks**: add a test that scans mock attempt payloads for `isCorrect` and explanation text before submit.
- **Per-section locking** must be enforced on the server, not just hidden in the UI.
