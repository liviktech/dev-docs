# Stats Developer Guide: Required Fields & Implementation

> **Target Audience:** Developers and AI Agents.  
> **Purpose:** Detailed reference for the Student Stats and Performance Dashboard feature across Mobile App, Backend, Web App, and Admin Panel, covering aggregate accuracy tracking, subject mastery breakdown, session history, paginated review queues, code paths, APIs, and database schemas.

---

## 1. Required Fields & Originating Modules

The Stats feature analyzes a student's practice and mock exam history to provide global performance metrics, subject-level accuracy breakdowns, test histories, and review queues for incorrect and bookmarked questions.

### 1.1 Progress Dashboard Metrics
*Delivered by `GET /api/v1/student/stats` and displayed on Mobile Stats tab and Web Stats page.*

| Field | Required | Where to Fill in App | Database Column | Notes |
|---|---|---|---|---|
| **Attempted** | Auto-calculated | Backend practice aggregation | Count from `assessment_attempt_items` | Total questions answered across submitted sessions. |
| **Correct** | Auto-calculated | Backend practice aggregation | Count from `assessment_attempt_items` | Total questions where `is_correct = true`. |
| **Global Accuracy** | Auto-calculated | Frontend format helper | Computed (`correct / attempted`) | Percentage rounded to integer (or dash if 0 attempted). |
| **Subjects Breakdown**| Auto-calculated | Backend join query | Array of subject statistics | Lists only subjects where student has answered at least 1 question. |
| **Subject Attempted** | Auto-calculated | Backend join query | Count of items per subject | Questions attempted in that taxonomy section. |
| **Subject Correct** | Auto-calculated | Backend join query | Count of correct items per subject | Correct questions in that taxonomy section. |
| **Session History** | Auto-derived | Recent attempts query | Last 20 completed sessions | Array of `{ id, config, score, total, submittedAt }`. |

---

### 1.2 Paginated Question Review Queues
*Delivered by `GET /api/v1/student/stats/questions?kind=incorrect|bookmarked&page=1&pageSize=10`.*

| Field | Required | Where to Fill in App | Database Column | Notes |
|---|---|---|---|---|
| **Kind** | Yes | Review Tab toggle | Query parameter | Must be either `'incorrect'` or `'bookmarked'`. |
| **Page** | Yes | Pagination / Show more | Query parameter | 1-based page index. |
| **Page Size** | Yes | Pagination / Show more | Query parameter | Number of items per page (default 10). |
| **Question ID** | Auto-derived | Database projection | `questions.id` | Identifier of the question. |
| **Subject Name** | Auto-derived | Taxonomy join | `taxonomy_nodes.name` | Display name of the parent subject. |
| **Topic Name** | Auto-derived | Taxonomy join | `taxonomy_nodes.name` | Display name of the question's primary topic. |
| **Stem** | Auto-derived | Question revision | `question_revisions.stem` | Markdown text of question. |
| **Your Answer** | Conditional | Distinct latest item answer | `question_options.text` | Text of chosen option (populated for `incorrect`; `null` for `bookmarked`). |
| **Correct Answer** | Conditional | Revision correct option | `question_options.text` | Text of correct option (populated for `incorrect`; `null` for `bookmarked`). |

---

### 1.3 Mock Exam History
*Delivered by `GET /api/v1/student/mock-exams/history` and listed in the Stats view.*

| Field | Required | Where to Fill in App | Database Column | Notes |
|---|---|---|---|---|
| **Attempt ID** | Auto-derived | Database query | `assessment_attempts.id` | Unique attempt identifier. |
| **Exam Name** | Auto-derived | Template join | `assessment_templates.title` | Name of the mock exam paper. |
| **Status** | Auto-derived | Database record | `assessment_attempts.status` | Either `'submitted'` or `'expired'`. |
| **Score / Percentage**| Auto-calculated | Database score record | `assessment_attempts.raw_score` | Correct count, total questions, and percentage. |
| **Submitted At** | Auto-derived | Database record | `assessment_attempts.submitted_at` | Timestamp of completion. |
| **Is Latest** | Auto-calculated | Window ranking | Boolean | `true` if this attempt is the student's most recent attempt for that exam paper. |

---

## 2. Eligibility & Validation Rules Checklist

Before assembling stats or returning question review queues, the system verifies:
1. **Student authorization guard:** Routes are locked under `StudentRoleGuard`. Parents and administrators receive 403.
2. **Dynamic subject suppression:** The `subjects` array only returns taxonomy sections where `attempted > 0`. Subjects without attempted questions are omitted to prevent empty charts.
3. **Latest-answer deduplication on incorrect list:**
   * Review queries use `DISTINCT ON (item.question_id) ... ORDER BY item.question_id, attempt.submitted_at DESC`.
   * If a student initially failed a question but subsequently answered it correctly during a retry session, the question is excluded from the incorrect review queue.
   * Questions are projected using the exact revision the student answered.
4. **Answer concealment on bookmarked list:**
   * For `kind = 'bookmarked'`, `yourAnswer` and `correctAnswer` are explicitly returned as `null` so students can re-test themselves without seeing the answers in advance.
5. **Strict pagination schema:** The review list enforces `z.enum(['incorrect', 'bookmarked'])`. Invalid `kind` parameters return 400 (`BAD_REQUEST`).
6. **Shared practice service reuse:** `StatsService` delegates history retrieval to `PracticeService.finishedSessions()`, preventing query discrepancies between the Home, Tests, and Stats tabs.

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
│    GET /api/v1/student/stats (Overview & Subjects)     │
│    GET /api/v1/student/stats/questions (Review Queue)  │
│    GET /api/v1/student/mock-exams/history              │
└───────────────────────────┬────────────────────────────┘
                            │
                            ▼
┌────────────────────────────────────────────────────────┐
│           PostgreSQL Database (Neon DB)                │
│   tables: assessment_attempts, assessment_attempt_     │
│   items, question_bookmarks, taxonomy_nodes,           │
│   question_revisions, question_options                 │
└────────────────────────────────────────────────────────┘
```

### 3.1 Mobile App (`src/`)
Implements the comprehensive Stats tab screen (`(tabs)/stats.tsx`). Displays the performance hero card with overall accuracy percentage and question count (`summary-card.tsx`), colored horizontal subject performance bars (`subject-performance.tsx`), chronologically ordered test history rows that tap to open past results (`test-history.tsx`), and tabbed review lists for incorrect and bookmarked questions with "Show more" pagination (`question-lists.tsx`).

* **Code References:**
  * Stats screen: `src/app/(tabs)/stats.tsx` (lines 25–150)
  * Summary hero card: `src/components/dashboard/summary-card.tsx` (lines 12–42)
  * Subject accuracy bars: `src/components/dashboard/subject-performance.tsx` (lines 15–60)
  * Test history list: `src/components/dashboard/test-history.tsx` (lines 20–80)
  * Question review list component: `src/components/dashboard/question-lists.tsx` (lines 25–105)
  * Stats API service: `src/services/stats-api.ts` (lines 10–38)
  * Stats React Query hooks: `src/hooks/use-stats.ts` (lines 10–55)
  * Format & styling utilities: `src/utils/stats-format.ts` (lines 1–35)
* **API:** `GET /student/stats`, `GET /student/stats/questions`, `GET /student/mock-exams/history`
* **DB Table:** `assessment_attempts`, `assessment_attempt_items`, `question_bookmarks`, `taxonomy_nodes`

---

### 3.2 Backend (`learning-app-backend`)
Provides performance tracking under `StatsModule`. `StatsController` exposes endpoints for aggregate stats and paged question lists. `StatsService.getStats` executes database aggregations over `assessment_attempt_items` joined to `taxonomy_nodes` and combines them with `PracticeService.finishedSessions(userId, 20)`. `getReviewQuestions` utilizes parameterised SQL queries with `DISTINCT ON` to page through recent mistakes or bookmarked questions.

* **Code References:**
  * Stats controller: `src/modules/stats/stats.controller.ts` (lines 20–85)
  * Stats service: `src/modules/stats/stats.service.ts` (lines 15–38)
  * Stats database repository: `src/modules/stats/repositories/stats.repository.ts` (lines 25–140)
  * Practice service history reuse: `src/modules/practice/practice.service.ts` (lines 275–310)
  * Stats integration tests: `test/integration/practice.int-spec.ts` (lines 240–310)
* **API:** `GET /api/v1/student/stats`, `GET /api/v1/student/stats/questions`, `GET /api/v1/student/mock-exams/history`
* **DB Table:** `assessment_attempts`, `assessment_attempt_modules`, `assessment_attempt_items`, `question_bookmarks`, `taxonomy_nodes`, `question_revisions`, `question_options`

---

### 3.3 Web App (`learning-web-app`)
Implements the web analytics page at `/stats`. Renders summary cards with overall accuracy and questions answered, subject performance progress bars with theme colors, a practice test history table, a mock exam history section (`mock-exam-history.tsx`) with direct links to past exam reviews, and tabbed review lists for incorrect and bookmarked questions with pagination.

* **Code References:**
  * Stats page: `src/app/(app)/stats/page.tsx` (lines 15–50)
  * Summary card: `src/components/stats/summary-card.tsx` (lines 12–40)
  * Subject performance: `src/components/stats/subject-performance.tsx` (lines 15–55)
  * Practice test history: `src/components/stats/test-history.tsx` (lines 18–65)
  * Mock exam history: `src/components/stats/mock-exam-history.tsx` (lines 15–60)
  * Review question lists: `src/components/stats/question-lists.tsx` (lines 20–100)
  * Web stats API & hooks: `src/lib/stats/api.ts` (lines 10–35), `src/lib/stats/hooks.ts` (lines 10–50)
* **API:** `/api/student/stats`, `/api/student/stats/questions`, `/api/student/mock-exams/history`
* **DB Table:** `assessment_attempts`, `assessment_attempt_items`, `question_bookmarks`

---

### 3.4 Admin Panel (`learning-app-admin`)
The admin panel authors and organizes the taxonomy subjects and category hierarchy that define the performance groupings on student stats pages. Question difficulty and taxonomy associations configured by staff determine how student responses are classified and scored.

* **Code References:**
  * Subjects & categories hierarchy: `src/features/content/pages/subjects-page.tsx` (lines 25–95)
* **API:** `GET /api/v1/admin/taxonomy`
* **DB Table:** `taxonomy_nodes`

---

## 4. What Already Works (Zero Changes Needed)

1. **Self-cleaning mistake review:** Once a student re-tests an incorrect question and answers it correctly, the `DISTINCT ON` SQL filter automatically drops that question from subsequent incorrect reviews.
2. **Unified history pipeline:** Stats and Home share identical `PracticeService` routines, ensuring session counts, timestamps, and scores remain perfectly synchronized across screens.
3. **Selective subject reporting:** Students only see progress metrics for subjects they have actually practiced, keeping the UI uncluttered.
4. **Blind bookmark review mode:** Bookmarked review cards hide answers and explanations, providing an authentic self-test experience.
5. **Cross-device invalidation:** Completing a session on mobile immediately updates the stats dashboard on web upon tab focus.
