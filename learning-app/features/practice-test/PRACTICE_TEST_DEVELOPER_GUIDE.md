# Practice Test Developer Guide: Required Fields & Implementation

> **Target Audience:** Developers and AI Agents.  
> **Purpose:** Detailed reference for the Practice Test feature across Mobile App, Backend, Web App, and Admin Panel, covering session configuration, question delivery, instant feedback, bookmarking, retry workflows, code paths, APIs, and database schemas.

---

## 1. Required Fields & Originating Modules

Practice Tests allow students to create customized, untimed or timed question sessions filtered by subject, topic, and difficulty, with instant answer feedback and bookmarking.

### 1.1 Session Configuration & Start
*Captured on the Practice Setup screen in Mobile and Web apps.*

| Field | Required | Where to Fill in App | Database Column | Notes |
|---|---|---|---|---|
| **Mode** | Yes | Practice Setup (`/practice/setup`) | `assessment_attempts.configuration->>'mode'` | Allowed: `'custom'`, `'bookmarked'`, `'incorrect'`. |
| **Subject ID** | Yes (for custom) | Subject selector | `assessment_attempt_modules.taxonomy_section_id` | UUID of active taxonomy section enrolled by student. |
| **Category IDs** | Optional | Topic multiselect | `assessment_attempts.configuration->'categoryIds'` | UUID array of domain/topic nodes under chosen subject. |
| **Difficulty** | Optional | Difficulty pills | `assessment_attempts.configuration->>'difficulty'` | Allowed: `1` (easy), `2` (medium), `3` (hard), or `null` (all). |
| **Question Count** | Yes | Count selector (`10`, `20`, custom) | `assessment_attempts.configuration->>'count'` | Integer between 1 and 20 (defaults to 10). |
| **Timed** | Optional | Timed toggle | `assessment_attempts.configuration->>'timed'` | Boolean (default `false`). |
| **Duration Minutes** | Conditional | Duration slider / chips | `assessment_attempt_modules.duration_seconds` | Required when `timed = true` (5 to 180 minutes). |

---

### 1.2 Answering Questions
*Captured on the Question Runner screen during an active practice session.*

| Field | Required | Where to Fill in App | Database Column | Notes |
|---|---|---|---|---|
| **Selected Option ID** | Yes | Option card / radio | `assessment_attempt_items.selected_option_id` | UUID referencing `question_options.id`. Single submission only. |
| **Is Correct** | Auto-calculated | Backend evaluation | `assessment_attempt_items.is_correct` | Evaluated against `question_options.is_correct` on submit. |
| **Answer Timestamp** | Auto-set | Server timestamp | `assessment_attempt_items.answered_at` | Set to `NOW()` when the answer is recorded. |

---

### 1.3 Bookmarking Questions
*Captured via the Bookmark button on the Question Runner screen or result view.*

| Field | Required | Where to Fill in App | Database Column | Notes |
|---|---|---|---|---|
| **User ID** | Auto-set | Authenticated student token | `question_bookmarks.user_id` | Enforces tenant-isolated bookmarks. |
| **Question ID** | Yes | Runner header / card | `question_bookmarks.question_id` | UUID referencing `questions.id`. Idempotent toggle. |

---

## 2. Eligibility & Validation Rules Checklist

Before initiating or updating a practice session, the system verifies:
1. **Student role verification:** The caller must possess the `student` role (`StudentRoleGuard`). Parents and administrators are rejected with 403.
2. **Enrolled subject check:** In `custom` mode, the specified `subjectId` must be in the student's active onboarding subjects (`student_subjects`). Unenrolled or invalid subjects return 422 (`INVALID_SUBJECT`).
3. **Active exam version:** An active `exam_versions` record must exist for the target exam (SAT or ACT). If missing, session initiation returns 503 (`PRACTICE_NOT_AVAILABLE`).
4. **Question availability threshold:** The database must contain at least 1 published question revision matching the selected criteria. If 0 matching questions are found, the endpoint returns 422 (`PRACTICE_NO_QUESTIONS`).
5. **No answer leaks:** Question payloads delivered to clients contain `answer: null`, and options omit `isCorrect`. Correct answers and explanations are returned exclusively in the response to an answer submission.
6. **Immutable answer submissions:** Answers are saved via `UPDATE ... WHERE selected_option_id IS NULL`. Submitting an answer to an already answered question returns 409 (`ALREADY_ANSWERED`).
7. **Session lifecycle guard:** Submitting an answer to a session whose status is already `'submitted'` returns 409 (`SESSION_SUBMITTED`).
8. **Option validity:** The submitted `optionId` must belong to the active revision of the question item. Foreign option IDs return 422 (`INVALID_OPTION`).
9. **Stable shuffling:** Questions and their corresponding options are shuffled once using Fisher-Yates (`shuffleArray`) at session creation and persisted to `assessment_attempt_items.position` and `option_order`. Resuming or refreshing maintains exact option order.
10. **Passage group preservation:** Questions tied to a shared reading passage (`groupKey`) remain contiguous in their original passage-internal order while the passage blocks shuffle freely.
11. **Order-dependent option protection:** Questions with option text matching order-dependent phrases ("All of the above", "None of the above", "Both A and B") or flagged `shuffle_options = false` retain their authoring order.

---

## 3. Four Sections Implementation

```text
┌────────────────────────────────────────────────────────┐
│           Mobile App / Web App / Admin Panel           │
└───────────────────────────┬────────────────────────────┘
                            │
                            ▼
┌────────────────────────────────────────────────────────┐
│               NestJS Backend Controller                │
│    GET /api/v1/student/practice/catalog                │
│    POST /api/v1/student/practice/sessions              │
│    GET /api/v1/student/practice/sessions/:id           │
│    PUT /api/v1/student/practice/sessions/:id/.../answer│
│    POST /api/v1/student/practice/sessions/:id/submit   │
│    GET /api/v1/student/practice/summary                │
│    PUT/DELETE /api/v1/student/bookmarks/:questionId    │
└───────────────────────────┬────────────────────────────┘
                            │
                            ▼
┌────────────────────────────────────────────────────────┐
│           PostgreSQL Database (Neon DB)                │
│   tables: assessment_attempts, assessment_attempt_     │
│   modules, assessment_attempt_items, question_         │
│   bookmarks, questions, question_revisions,            │
│   question_options, taxonomy_nodes, exam_versions      │
└────────────────────────────────────────────────────────┘
```

### 3.1 Mobile App (`src/`)
Provides a complete mobile practice experience including custom setup, active question running with immediate answer feedback, passage reading, bookmarking, and result review. Students access practice via the Tests tab (`tests.tsx`), configure criteria in `setup.tsx`, and run sessions in `[sessionId].tsx`. Answers update local query state, display explanations immediately, and lock options against re-selection. Finished sessions navigate to `result/[sessionId].tsx` for performance breakdown by topic and one-tap "Retry incorrect questions".

* **Code References:**
  * Tests tab entry: `src/app/(tabs)/tests.tsx` (lines 35–95)
  * Practice setup screen: `src/app/practice/setup.tsx` (lines 30–175)
  * Practice runner screen: `src/app/practice/[sessionId].tsx` (lines 35–190)
  * Practice result screen: `src/app/practice/result/[sessionId].tsx` (lines 25–95)
  * Practice API service: `src/services/practice-api.ts` (lines 101–113)
  * Practice hook: `src/hooks/use-practice.ts` (lines 15–110)
  * Quick session hook: `src/hooks/use-quick-session.ts` (lines 10–35)
* **API:** `GET /student/practice/catalog`, `POST /student/practice/sessions`, `GET /student/practice/sessions/:id`, `PUT /student/practice/sessions/:id/items/:itemId/answer`, `POST /student/practice/sessions/:id/submit`, `GET /student/practice/summary`, `PUT /student/bookmarks/:questionId`, `DELETE /student/bookmarks/:questionId`
* **DB Table:** `assessment_attempts`, `assessment_attempt_modules`, `assessment_attempt_items`, `question_bookmarks`, `question_revisions`, `taxonomy_nodes`, `exam_versions`

---

### 3.2 Backend (`learning-app-backend`)
Implements the practice engine under `PracticeModule`, enforcing strict Zod validation, question candidate gathering, Fisher-Yates shuffling, single-attempt locking, and instant feedback evaluation. `PracticeService.createSession` checks enrolled student subjects, queries available questions prioritizing unanswered ones, randomizes questions and options, and creates the attempt inside a database transaction. `answerQuestion` verifies option ownership, records `selected_option_id`, and returns `isCorrect`, `correctOptionId`, and `explanation`.

* **Code References:**
  * Practice controller: `src/modules/practice/practice.controller.ts` (lines 45–280)
  * Practice service: `src/modules/practice/practice.service.ts` (lines 55–440)
  * Practice repository: `src/modules/practice/repositories/practice.repository.ts` (lines 30–425)
  * Request validation schemas: `src/modules/practice/schemas/practice-request.schemas.ts` (lines 10–50)
  * Response documentation schemas: `src/modules/practice/schemas/practice-response.schemas.ts` (lines 15–125)
  * Assessment clock utility: `src/shared/assessment/clock.ts` (lines 1–28)
  * Assessment shuffle utility: `src/shared/assessment/shuffle.ts` (lines 1–76)
* **API:** `GET /api/v1/student/practice/catalog`, `POST /api/v1/student/practice/sessions`, `GET /api/v1/student/practice/sessions/:id`, `PUT /api/v1/student/practice/sessions/:id/items/:itemId/answer`, `POST /api/v1/student/practice/sessions/:id/submit`, `GET /api/v1/student/practice/summary`, `PUT /api/v1/student/bookmarks/:questionId`, `DELETE /api/v1/student/bookmarks/:questionId`
* **DB Table:** `assessment_attempts`, `assessment_attempt_modules`, `assessment_attempt_items`, `question_bookmarks`, `questions`, `question_revisions`, `question_options`, `taxonomy_nodes`, `exam_versions`

---

### 3.3 Web App (`learning-web-app`)
Implements an Acely-style split-pane web practice interface with full keyboard accessibility (A–E for options, arrows for navigation). `/practice` provides resume prompts, quick start presets, and direct links to bookmarked and incorrect sessions. `/practice/setup` configures subjects, domains, and question counts. `/practice/[sessionId]` renders the full-screen runner with independent scrolling for reading passages and question options, updating answers and bookmarks in real time via TanStack Query mutations.

* **Code References:**
  * Practice dashboard page: `src/app/(app)/practice/page.tsx` (lines 15–40)
  * Practice setup page: `src/app/(app)/practice/setup/page.tsx` (lines 25–155)
  * Full-screen runner page: `src/app/(focus)/practice/[sessionId]/page.tsx` (lines 20–80)
  * Result page: `src/app/(app)/practice/[sessionId]/result/page.tsx` (lines 20–68)
  * Question runner component: `src/components/practice/question-runner.tsx` (lines 30–175)
  * Client API module: `src/lib/practice/api.ts` (lines 20–95)
  * React Query hooks: `src/lib/practice/hooks.ts` (lines 25–115)
* **API:** `/api/student/practice/catalog`, `/api/student/practice/sessions`, `/api/student/practice/sessions/:id`, `/api/student/practice/sessions/:id/items/:itemId/answer`, `/api/student/practice/sessions/:id/submit`, `/api/student/practice/summary`, `/api/student/bookmarks/:id`
* **DB Table:** `assessment_attempts`, `assessment_attempt_modules`, `assessment_attempt_items`, `question_bookmarks`

---

### 3.4 Admin Panel (`learning-app-admin`)
The admin panel provides the authoring pipeline that feeds the question bank used by practice tests. Administrators curate questions, edit markdown stems and explanations, configure option choices, assign taxonomy subjects and categories, and toggle option shuffling (`shuffleOptions`). Questions created or imported via the admin panel become immediately available in student practice catalogs once published.

* **Code References:**
  * Question catalog page: `src/features/content/pages/questions-page.tsx` (lines 25–175)
  * Question form and editor: `src/features/content/pages/question-form-page.tsx` (lines 30–215)
  * Bulk question import page: `src/features/content/pages/import-page.tsx` (lines 20–145)
  * Taxonomy subjects management: `src/features/content/pages/subjects-page.tsx` (lines 25–95)
  * Content API client: `src/features/content/api/content.api.ts` (lines 35–110)
* **API:** `GET /api/v1/admin/questions`, `POST /api/v1/admin/questions`, `PATCH /api/v1/admin/questions/:id`, `POST /api/v1/admin/questions/bulk`, `GET /api/v1/admin/taxonomy`
* **DB Table:** `questions`, `question_revisions`, `question_options`, `taxonomy_nodes`, `passages`

---

## 4. What Already Works (Zero Changes Needed)

1. **Zero question leaks:** API contracts guarantee that unresolved question objects never include `isCorrect` or explanation fields before student submission.
2. **Deterministic attempt shuffling:** Options and questions are randomized using Fisher-Yates at creation and stored in `option_order` and `position`, ensuring stability across client page reloads and reviews.
3. **Double-submit race protection:** Answers are written using a single atomic SQL condition (`WHERE selected_option_id IS NULL`), rejecting concurrent submissions with 409 `ALREADY_ANSWERED`.
4. **Smart question selection:** The question selector prioritizes unseen questions over previously answered questions, automatically rotating practice content.
5. **Dynamic retry resolution:** The `incorrect` mode dynamically checks the student's latest response per question; once a question is answered correctly in retry, it disappears from subsequent retry queues.
6. **Passage integrity guarantee:** Shuffling preserves passage coherence by grouping questions linked to the same reading passage together in their intended sequential order.
