# Plan: Practice Tests, End to End (Mobile + Backend + Database)

## Status (2026-11-19): code written, not yet run on the device

- Done: migration 0010, seed `0002_exam-versions.sql`, the `practice` backend module (all 8 routes), 21 integration tests and 5 unit tests (full backend suite: 158 integration + 57 unit pass; typecheck and lint clean), `architecture.md` section 10e, the mobile data layer, components, setup / question / result screens, the new Tests tab, and removal of the mock questions and `PracticeProvider`. Mobile `tsc` and `expo lint` are clean.
- Differences from this plan: the topic and difficulty counts are in the catalog as `easy / medium / hard`; `submit` and `GET session` share one reply shape (the session with its `result`); the bookmarked and retry cards start a session of up to 20 questions.
- Options keep the admin's order (no server shuffle). `architecture.md` used to say mobile endpoints must shuffle; that line was changed to match.
- Not yet done: migration 0010 and seed 0002 are applied to the **test** database only, not Neon dev. The manual checklist in section 8 has not been run on the emulator.

Date: 2026-11-19
Apps: Mobile app (`D:\lm`) and Backend (`D:\Livik-projects\Learning-app\learning-app-backend`)
Screen: the **Tests** tab and the practice question screens.

## 1. Goal

Today the Tests tab opens one fixed practice test of 5 hard-coded questions. Nothing is saved.

After this work, a student can:

1. **Choose what to practice**: a subject, optionally some topics, a difficulty, and how many questions.
2. **See instant feedback**: right answer shows its explanation. Wrong answer highlights the correct option and shows the same explanation.
3. **Move next and previous** between questions.
4. **Bookmark** a question to come back to it, and **retry the ones they got wrong**.

Everything comes from the database through the backend. Nothing is hard-coded in the app.

## 2. What is already there

### Mobile app (`D:\lm`)

| Item | Where | Status |
|---|---|---|
| Tests tab with one "Practice tests" card | `src/app/(tabs)/tests.tsx` | Works, but static |
| Question screen: letters A-D, green/red colouring, flash, shake and bounce animation, progress bar, "Correct!" message | `src/app/practice/[index].tsx` | **Reuse** the look and animations |
| Answer locks after the first tap | `practice-context.tsx` (`q in prev ? prev : ...`) | **Keep** this rule |
| Score kept in memory | `src/state/practice-context.tsx` | Replace (lost on app restart) |
| Fake questions (5), no explanation, no subject, no difficulty | `src/data/mock-practice.ts` | **Delete** |
| API client with token refresh (`apiGet`, `apiPut`, `apiAuthedPost`) | `src/services/api-client.ts` | Reuse. Needs `apiDelete` |
| React Query set up | `src/app/_layout.tsx` | Reuse |
| Subject list for the student (from onboarding) | `src/services/onboarding-api.ts`, `use-onboarding.ts` | Reuse |
| Small components: `PrimaryButton`, `ProgressBar`, `Chip`, `ThemedText`, `ThemedView` | `src/components/` | Reuse. Add a `secondary` variant to `PrimaryButton` |
| Bookmarked / Incorrect list UI (shows mock data on the Stats screen) | `src/components/dashboard/question-lists.tsx` | Not part of this work |

### Backend

| Item | Status |
|---|---|
| Questions, revisions, options, **explanation**, difficulty 1-3 (easy, medium, hard), passage, subject and category | Done, but only for admins (`/admin/questions`). Admin responses show the correct answer, so the app must not use them |
| Taxonomy: subject (`section`) and category (`domain`, shown as "topic" to the student) | Done |
| Student guard (`StudentRoleGuard`) and student module for onboarding | Done. The `student_subjects` table holds the subjects the student picked |
| Tables `assessment_attempts` (type `practice_session`, with a `configuration` JSON for the filters), `assessment_attempt_modules`, `assessment_attempt_items` (selected option, `is_correct`, `is_flagged`, `time_spent_seconds`) | **Already in migration 0001, never used.** We reuse them, no new answer tables |
| Table `question_bookmarks (user_id, question_id)` | **Already exists, never used** |
| **Any student-facing question API** | **Missing** |

### Gaps to close

- No API that gives a student questions, checks an answer, or scores a session.
- No bookmark API. No "incorrect questions" query.
- `exam_versions` has **no seed data** (`seeds/0001` only has exams and taxonomy), and `assessment_attempts.exam_version_id` is required. Needs a seed row per exam.
- `assessment_attempt_modules.duration_seconds` must be > 0, but practice has no timer. Needs a small migration.
- The app has no setup screen, no previous button, no bookmark button, no explanation view, no result screen.

## 3. Decisions (change any of these before we start)

| # | Decision | Why |
|---|---|---|
| 1 | One session = **one subject**. Topics (categories) are optional and can be several. Difficulty is Easy / Medium / Hard / **Mixed** (default). | Matches the attempt table (one module has one subject). A mixed-subject session can come later |
| 2 | The correct answer is **never sent with the question**. It comes back only in the reply to the student's answer. | Otherwise the answer is visible in the network traffic before the student answers |
| 3 | An answer is **final** once sent (same as today). | Keeps scoring honest. Retry is done through a new session |
| 4 | **Next is always enabled**, so a question can be skipped. Skipped questions count as not answered. | Bookmark is for "answer later", so skipping must be possible. Easy to switch to "must answer first" |
| 5 | "Retry incorrect" starts a **new session** made of the questions whose latest answer is wrong. | Keeps the old session's score unchanged |
| 6 | Question text is plain text in version 1. Markdown, formulas and images are a later step. | The app has no markdown renderer yet. Needs a check with real content (see risks) |
| 7 | Question order is random. No timer. | Practice mode, not an exam |

## 4. Backend work

New NestJS module `practice` (`src/modules/practice/`), same layering as `student/`: controller, service, repository, zod schemas, DTOs, interfaces. All routes use `StudentRoleGuard`. The student id comes from the token, never from the request.

### 4.1 Database

- **Migration `0010_practice-sessions.sql`** (needed; migrations 0001-0009 are already applied everywhere)
  - Make `assessment_attempt_modules.duration_seconds` nullable and change its check to "null or > 0". Mock exams still need it.
  - Index on `assessment_attempt_items (question_revision_id)` already exists. Add one on `assessment_attempts (user_id, assessment_type, status)` for "resume" and history.
- **Seed**: add one active `exam_versions` row for SAT and one for ACT (new seed file, same style as `0001_exams-and-taxonomy.sql`, safe to run twice). Columns: `exam`, `code`, `name`, `effective_from`, `is_active`. See risk 7 for why.
- No new tables. Sessions, answers and bookmarks use the existing ones.

### 4.2 API

All under `/api/v1/student/practice` (and `/student/bookmarks`). All replies use the existing success envelope.

| # | Route | What it does |
|---|---|---|
| 1 | `GET /student/practice/catalog` | For the setup screen. The student's chosen subjects, each with its topics, and how many **published** questions exist per difficulty. Lets the app cap the question count and hide empty choices |
| 2 | `POST /student/practice/sessions` | Body: `{ mode, subjectId?, categoryIds?, difficulty?, count }`. `mode` is `custom`, `bookmarked` or `incorrect`. Picks random published, non-archived questions, creates the attempt, one module and the items (pinned to the exact revision). Returns the session and its questions **without answers**. If fewer questions exist than asked, it returns what exists. If there are none, `422` with a clear message |
| 3 | `GET /student/practice/sessions/:id` | Same shape as the create reply, plus for each answered question: chosen option, `isCorrect`, `correctOptionId`, `explanation`, and `bookmarked`. Used to resume, to go Previous, and for the result review |
| 4 | `PUT /student/practice/sessions/:id/items/:itemId/answer` | Body: `{ optionId }`. Saves the answer and returns `{ isCorrect, correctOptionId, explanation }`. A second answer for the same item is `409`. Answering a submitted session is `409` |
| 5 | `POST /student/practice/sessions/:id/submit` | Marks the session submitted, stores `raw_score`, returns `{ score, total, answered, skipped, byTopic[] }`. Calling it twice returns the same result |
| 6 | `GET /student/practice/summary` | For the Tests tab: bookmarked count, incorrect-to-retry count, the unfinished session (if any) and the last few finished sessions with their score |
| 7 | `PUT /student/bookmarks/:questionId` | Adds a bookmark (safe to repeat) |
| 8 | `DELETE /student/bookmarks/:questionId` | Removes it (safe to repeat) |

What a question looks like in a reply: `itemId`, `questionId`, `position`, `stem`, `passage` (text or null), `difficulty`, `topic`, `options[]` as `{ id, key, text }`, `bookmarked`. There is **no** `isCorrect` on options.

### 4.3 Rules in the code

- **Picking questions**: question must be not archived and have a published revision. Filter by subject, categories and difficulty. For `bookmarked` use the student's bookmarks. For `incorrect` use questions whose **latest** answered item (by `updated_at`) has `is_correct = false`. Random order. Where there is a choice, prefer questions the student has not seen.
- **Answering**: check that the session belongs to the student, is not submitted, the item belongs to it, and the option belongs to that question revision. Set `selected_option_id`, `is_correct` and `answer_changed_at` in one statement, so a double tap cannot save two answers.
- **Submit**: score = count of `is_correct = true`. Set `status = submitted`, `submitted_at` (the table's check requires both).
- **Bookmark**: check that the question exists and is not archived.
- Student A can never read or answer student B's session (return `404`, not `403`).

### 4.4 Backend tests (Vitest, same style as `test/integration/question-upload.int-spec.ts`)

- Unit: question-pick filters, score calculation, "latest answer wins" for incorrect.
- Integration: create session (all three modes, not enough questions, no questions), answer right and wrong (explanation returned), double answer is `409`, submit twice, resume reply hides unanswered answers, bookmark add/remove twice, another student gets `404`, admin token gets `403`, **answers never appear in the create reply**.
- Run: `npm run typecheck && npm run lint && npm test && npm run test:integration`.
- Update `architecture.md` with a new section (like 10d) and the endpoint table.

## 5. Mobile work

Rule from the project skill `reusable-ui-components`: every UI piece is a reusable component, screens only compose them, theme tokens only, server data only through React Query hooks, API calls only in `src/services/`.

### 5.1 Data layer

| File | What |
|---|---|
| `src/services/api-client.ts` | Add `apiDelete` |
| `src/services/practice-api.ts` (new) | Types and one function per route in section 4.2 |
| `src/hooks/use-practice.ts` (new) | `usePracticeCatalog`, `usePracticeSummary`, `usePracticeSession(id)`, `useStartSession`, `useSubmitAnswer`, `useSubmitSession`, `useToggleBookmark`. After an answer or bookmark, update the cached session in place (no refetch); after submit, refresh the summary |
| `src/state/practice-context.tsx`, `src/data/mock-practice.ts` | **Delete.** Remove `PracticeProvider` from `src/app/_layout.tsx`. `profile.tsx` stops using `usePractice` and reads the last score from `usePracticeSummary` (logout clears the query cache instead) |

### 5.2 Screens (Expo Router)

| Route | What the student sees |
|---|---|
| `(tabs)/tests.tsx` (changed) | Title "Tests". Card **Start practice** → setup. Card **Continue** if a session is unfinished. Two small cards: **Bookmarked (n)** and **Retry incorrect (n)** (disabled when 0). **Recent sessions** with their scores |
| `practice/setup.tsx` (new) | 1) Subject (one of the student's). 2) Topics (chips, optional, "All topics" by default). 3) Difficulty: Easy / Medium / Hard / Mixed. 4) Number of questions (5, 10, 15, 20, capped by what exists; shows "n available"). Button **Start**. Shows an empty state if no questions match |
| `practice/[sessionId].tsx` (replaces `[index].tsx`) | The question screen. One screen that keeps the current index in state, so Previous and Next are instant. Details below |
| `practice/result/[sessionId].tsx` (new) | Score, answered and skipped, per-topic bar. Buttons: **Retry incorrect**, **Review answers**, **Back to Tests**. Review shows each question with the student's answer, the correct one and the explanation |
| `practice/_layout.tsx` | Add the new routes to the stack |

**Question screen behaviour**

- Top: "Question 3 of 10", progress bar, **bookmark icon** (filled when saved).
- Passage (if the question has one) in a scrollable card above the question.
- Tap an option: send the answer, then colour it (existing look). Wrong: chosen option red, correct option green. Both cases show the **explanation card** under the options. If a question has no explanation, show a short "No explanation available" line.
- Going **Previous** to an answered question shows it in the answered state (colours and explanation), locked.
- Footer: **Previous** (hidden on the first question), **Next**. On the last question **Next** becomes **Finish**. If some questions are unanswered, ask "N questions not answered. Finish anyway?".
- Leaving with the back gesture asks to confirm. The session stays saved and appears as **Continue** on the Tests tab.
- Keep the existing flash, shake and bounce animations.

### 5.3 New reusable components

Search `src/components/` first; these do not exist yet.

| Component | Used for |
|---|---|
| `practice/answer-option` | One option row: letter, text, idle / correct / wrong / locked states (moved out of the old screen) |
| `practice/explanation-card` | Explanation box with a correct/wrong heading |
| `practice/question-header` | "Question n of N" + `ProgressBar` + bookmark button |
| `practice/question-nav` | Previous / Next / Finish footer |
| `practice/passage-card` | Scrollable passage |
| `practice/practice-entry-card` | The cards on the Tests tab (start, continue, bookmarked, retry) |
| `practice/session-row` | One recent session with score and date |
| `practice/result-summary` | Score ring/number and answered / skipped counts |
| `practice/review-item` | One question in the review list |
| `ui/icon-button` | Bookmark and close buttons (44pt touch target, accessibility label) |
| `ui/choice-chip` | Selectable chip for subject, topics and difficulty (extends the look of `hub/chip`) |
| `ui/count-picker` | Question count choices |
| `ui/empty-state` and `ui/loading-state` | Empty and loading views |
| `PrimaryButton` | Add `variant: 'secondary'` and `loading` instead of making a new button |

### 5.4 Checks

- Mobile has no test runner. Required: `npx tsc --noEmit` and `npx expo lint` clean.
- Manual run on the emulator (see checklist in section 8).

## 6. Order of work

1. **Backend foundation**: migration 0010, exam-version seed, `practice` module with routes 1-5, tests.
2. **Backend extras**: bookmarks, summary, `bookmarked` and `incorrect` modes, tests, `architecture.md`.
3. **Mobile data layer**: `apiDelete`, `practice-api`, hooks, remove the context and mock data.
4. **Mobile screens**: setup → question screen (answer, explanation, previous/next) → result → Tests tab → bookmark and retry.
5. **Polish and check**: empty/error/offline states, tsc, lint, manual checklist, PR docs.

Steps 1-2 and 3 can be done by different people at the same time once the route shapes in 4.2 are agreed.

## 7. Risks and open points

- **Test data**: questions are already uploaded through the admin panel. At the start, check that each subject we test has published questions in more than one difficulty, so the filters can be tested.
- **Question text format**: admin questions are markdown and may contain formulas or images. Version 1 shows plain text. If the real content needs formulas, that is a next step (see section 9).
- **Question count**: a session asks for up to 20 questions in one reply. Fine for now; no paging.
- **Migrations**: 0001-0009 are already applied. Only the new `0010` has to be applied, locally and on Neon dev.
- **Exam version seed (needs action in every environment)**: each practice session is stored in `assessment_attempts`, which requires an `exam_version_id` pointing to a row in `exam_versions`. That table is created by migration 0001, but **nothing fills it** and the admin panel does not manage it. Questions do not need it, so uploading questions worked without it. Without the two rows, starting a session fails with a foreign key error. So: run the new seed once on local and on Neon dev (and later on production). The API also returns a clear error if an exam has no active version, instead of crashing.

## 8. Manual test checklist (emulator)

- [ ] Setup: only the student's subjects show; topics change with the subject; counts cap correctly; empty combination shows the empty state.
- [ ] Start 5 Easy questions: all are Easy and from the chosen subject.
- [ ] Right answer: green, explanation shows. Wrong answer: chosen red, correct green, explanation shows.
- [ ] Tapping a second option on the same question does nothing.
- [ ] Next, Previous: answered questions come back locked with colours and explanation.
- [ ] Skip a question, finish: confirm message appears, result counts it as skipped.
- [ ] Bookmark on, kill the app, reopen: still bookmarked. Bookmark off works.
- [ ] Tests tab: Bookmarked (n) starts a session with exactly those questions.
- [ ] Get 2 wrong, finish, **Retry incorrect**: new session has exactly those 2. Get them right, finish: Retry count goes down.
- [ ] Close the app mid-session: **Continue** resumes at the right place with earlier answers kept.
- [ ] Network off: a clear error, no crash, the question can be answered again after reconnecting.
- [ ] The network response for starting a session contains no correct answer.

## 9. Out of scope on purpose (listed as "Next steps" in the PR documents)

When the PRs are written after the implementation, copy this list into each PR's next-steps part.

- **Timed tests, mock exams and mixed-subject sessions.** Practice has no timer. Mock exams are a separate product (fixed question sets, modules, scoring). A session with several subjects needs one section per subject in the data.
- **Markdown, formula and image rendering.** Admin questions are saved as markdown. The app shows plain text for now, so formulas, tables and images look raw until a renderer is added.
- **Stats tab.** Its bookmarked and incorrect lists still show mock data. This work builds the real data; wiring that screen is a separate task.
- **Changing answers, hints and per-question time tracking.** Answers are final, there are no hints, and the `time_spent_seconds` column stays empty.
- **Editing questions or admin panel changes.** This work only reads existing questions. Question content, explanations and admin screens are not touched. A question without an explanation shows "No explanation available"; adding one is done in the admin panel.
