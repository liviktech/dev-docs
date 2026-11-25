# Mock Exam Developer Guide: Required Fields & Implementation

> **Target Audience:** Developers and AI Agents.  
> **Purpose:** Detailed reference for the Mock Exam feature across Mobile App, Backend, Web App, and Admin Panel, covering admin exam authoring, section configuration, timed student execution, server-owned clock enforcement, scoring, and post-submission review.

---

## 1. Required Fields & Originating Modules

Mock Exams provide simulated full-length exam papers (SAT and ACT) structured into subject sections with overall or per-section time limits, blind answer entry (no feedback during the test), and in-depth review post-submission.

### 1.1 Admin Mock Exam Paper Builder
*Captured in the Admin Panel Exam Editor (`/exams/new` and `/exams/:id/edit`).*

| Field | Required | Where to Fill in App | Database Column | Notes |
|---|---|---|---|---|
| **Exam Name** | Yes | Exam Details form | `assessment_templates.title`<br>`assessment_template_versions.title` | Text string (1 to 150 characters). |
| **Target Test** | Yes | Test select | `assessment_templates.exam_version_id` | Mapped to active SAT or ACT exam version. Cannot change after creation. |
| **Instructions** | Optional | Exam Details form | `assessment_template_versions.instructions` | Markdown guidance presented to student prior to start. |
| **Timing Mode** | Yes | Timing switcher | `assessment_template_versions.timing_mode` | Either `'overall'` (single countdown) or `'per_section'` (countdown per module). |
| **Total Duration** | Conditional | Duration presets / custom | `assessment_template_versions.total_duration_seconds` | Required when `timing_mode = 'overall'`. Stored in seconds (1 to 360 minutes). |
| **Shuffle Questions**| Yes | Settings toggle | `assessment_template_versions.shuffle_questions` | Boolean (default `true`). Shuffles question order inside sections. |
| **Sections** | Yes (min 1) | Section editor | `assessment_modules` | Ordered list of exam sections. |
| **Section Subject** | Yes | Section selector | `assessment_modules.taxonomy_section_id` | UUID of taxonomy subject node. Defines section display title. |
| **Section Duration** | Conditional | Section duration field | `assessment_modules.duration_seconds` | Required when `timing_mode = 'per_section'`. Stored in seconds. |
| **Question IDs** | Yes (min 1) | Question picker modal | `assessment_module_questions.question_revision_id` | UUID array of published question revisions. No duplicates allowed. |

---

### 1.2 Student Mock Exam Execution & Answering
*Captured in Mobile and Web exam runners during an active attempt.*

| Field | Required | Where to Fill in App | Database Column | Notes |
|---|---|---|---|---|
| **Attempt ID** | Auto-derived | Start / Resume route | `assessment_attempts.id` | Single in-progress attempt enforced per student and exam version. |
| **Item ID** | Yes | Exam runner | `assessment_attempt_items.id` | Identifies specific question item in the attempt. |
| **Selected Option ID**| Optional | Option selector | `assessment_attempt_items.selected_option_id` | UUID of chosen option. Can be set to `null` to clear selection. |
| **Marked for Review** | Optional | Flag button / star | `assessment_attempt_items.is_flagged` | Boolean flag for student reference on the question navigator. |
| **Section ID** | Conditional | Submit Section button | `assessment_attempt_modules.id` | Required when concluding a section early in `per_section` mode. |

---

### 1.3 Post-Submission Review & Scoring
*Calculated by the backend and displayed on Result and History screens.*

| Field | Required | Where to Fill in App | Database Column | Notes |
|---|---|---|---|---|
| **Raw Score** | Auto-set | Final submit / expiry | `assessment_attempts.raw_score` | Count of correctly answered questions. Unanswered counts as wrong. |
| **Percentage** | Auto-set | Final submit / expiry | Computed | `(raw_score / total_questions) * 100`, rounded to 1 decimal place. |
| **Subject Breakdown**| Auto-set | Final submit / expiry | Computed | Correct count and total questions grouped by section subject. |
| **Status** | Auto-set | Final submit / expiry | `assessment_attempts.status` | `'submitted'` if finished by student, or `'expired'` if time ran out. |

---

## 2. Eligibility & Validation Rules Checklist

Before creating, publishing, taking, or reviewing mock exams, the system verifies:
1. **Admin role requirement:** Admin exam management routes (`/api/v1/admin/mock-exams/*`) are restricted to active admin staff (`AdminRoleGuard`).
2. **Paper completeness before publish:** An exam paper cannot be published unless it has at least 1 section, every section has at least 1 question, and valid durations are specified. Failed validations return 422 (`EXAM_INVALID`).
3. **No duplicate questions:** A question revision cannot appear more than once within the same exam paper across all sections.
4. **Draft revision snapshotting:** Modifying a published exam creates a new draft version. Students with in-progress attempts continue on the version they started without disruption.
5. **Single in-progress attempt concurrency:** A unique partial index (`one_open_mock_attempt_idx`) ensures a student can have only one active attempt per exam version. Initiating an existing exam returns the active attempt idempotently.
6. **No answer leaks during execution:** The attempt endpoints (`GET /attempts/:id`) never deliver `isCorrect`, correct option IDs, or explanations.
7. **Strict section locking:** In `per_section` mode, answering questions in locked, completed, or upcoming sections is rejected on the database write.
8. **Server-owned clock enforcement:** The server computes `remainingSeconds` dynamically. Answers submitted after `expires_at + 5 seconds` (grace period) are rejected with 409 (`ATTEMPT_EXPIRED` or `SECTION_LOCKED`).
9. **Review route guard:** Access to `GET .../review` returns 409 (`REVIEW_NOT_AVAILABLE`) until the attempt transitions to `'submitted'` or `'expired'`.
10. **Tenant data isolation:** Mock exam attempts belonging to another student return 404 (`NOT_FOUND`), never 403.

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
│    GET /api/v1/admin/mock-exams (Admin Catalog)        │
│    POST /api/v1/admin/mock-exams (Create Draft)        │
│    POST /api/v1/admin/mock-exams/:id/publish           │
│    GET /api/v1/student/mock-exams (Student Catalog)    │
│    POST /api/v1/student/mock-exams/:id/attempts        │
│    GET /api/v1/student/mock-exams/attempts/:id         │
│    PUT /api/v1/student/mock-exams/attempts/:id/.../ans │
│    POST /api/v1/student/mock-exams/attempts/:id/submit │
│    GET /api/v1/student/mock-exams/attempts/:id/review  │
└───────────────────────────┬────────────────────────────┘
                            │
                            ▼
┌────────────────────────────────────────────────────────┐
│           PostgreSQL Database (Neon DB)                │
│   tables: assessment_templates, assessment_template_   │
│   versions, assessment_modules, assessment_module_     │
│   questions, assessment_attempts, assessment_attempt_  │
│   modules, assessment_attempt_items, question_revisio  │
└────────────────────────────────────────────────────────┘
```

### 3.1 Mobile App (`src/`)
Provides a mobile mock exam experience including catalog browsing under the Tests tab, timed exam execution with centered header countdown, numbered question palette, section completion dialogs, score result presentation, and answer review. `index.tsx` lists published mock exams. `[attemptId].tsx` hosts the exam runner (`exam-runner.tsx`) which shows questions one at a time, hides feedback, allows changing answers, and supports marking questions for review. Backgrounding is handled via React Native `AppState` to refresh clock deadlines upon returning.

* **Code References:**
  * Mock exam catalog screen: `src/app/mock-exams/index.tsx` (lines 20–95)
  * Exam runner screen shell: `src/app/mock-exams/[attemptId].tsx` (lines 15–45)
  * Exam result & review screen: `src/app/mock-exams/result/[attemptId].tsx` (lines 20–110)
  * Runner UI component: `src/components/mock-exam/exam-runner.tsx` (lines 30–185)
  * Header countdown timer: `src/components/mock-exam/exam-timer.tsx` (lines 15–80)
  * Question palette component: `src/components/mock-exam/question-palette.tsx` (lines 12–68)
  * Review item component: `src/components/mock-exam/exam-review-item.tsx` (lines 15–70)
  * Mock exam API client: `src/services/mock-exam-api.ts` (lines 140–202)
  * Mock exam query hooks: `src/hooks/use-mock-exam.ts` (lines 20–125)
  * Exam runner state hook: `src/hooks/use-exam-runner.ts` (lines 15–105)
* **API:** `GET /student/mock-exams`, `POST /student/mock-exams/:examId/attempts`, `GET /student/mock-exams/attempts/:id`, `PUT /student/mock-exams/attempts/:id/items/:itemId/answer`, `POST /student/mock-exams/attempts/:id/sections/:sectionId/submit`, `POST /student/mock-exams/attempts/:id/submit`, `GET /student/mock-exams/attempts/:id/review`, `GET /student/mock-exams/history`
* **DB Table:** `assessment_templates`, `assessment_template_versions`, `assessment_modules`, `assessment_attempts`, `assessment_attempt_modules`, `assessment_attempt_items`

---

### 3.2 Backend (`learning-app-backend`)
Provides two cooperating modules: `ExamsModule` for admin paper management and `MockExamsModule` for student execution. `ExamsService` validates exam structures, enforces timing parameters, and snapshots revisions upon publishing. `MockExamsService` initializes attempts, applies Fisher-Yates question and option shuffling, locks closed sections, scores attempts, and provides the post-completion review. The server clock handles lazy expiry on every read/write via `settle()`.

* **Code References:**
  * Admin exams controller: `src/modules/exams/exams.controller.ts` (lines 40–250)
  * Admin exams service: `src/modules/exams/exams.service.ts` (lines 50–300)
  * Admin exams repository: `src/modules/exams/repositories/exams.repository.ts` (lines 30–375)
  * Student mock exams controller: `src/modules/mock-exams/mock-exams.controller.ts` (lines 45–290)
  * Student mock exams service: `src/modules/mock-exams/mock-exams.service.ts` (lines 60–485)
  * Student mock exams repository: `src/modules/mock-exams/repositories/mock-exams.repository.ts` (lines 35–415)
  * Assessment clock utilities: `src/shared/assessment/clock.ts` (lines 1–28)
  * Assessment shuffle utilities: `src/shared/assessment/shuffle.ts` (lines 1–76)
* **API:** `GET /api/v1/admin/mock-exams`, `POST /api/v1/admin/mock-exams`, `GET /api/v1/admin/mock-exams/:id`, `PATCH /api/v1/admin/mock-exams/:id`, `POST /api/v1/admin/mock-exams/:id/publish`, `POST /api/v1/admin/mock-exams/:id/unpublish`, `DELETE /api/v1/admin/mock-exams/:id`, `GET /api/v1/student/mock-exams`, `POST /api/v1/student/mock-exams/:id/attempts`, `GET /api/v1/student/mock-exams/attempts/:id`, `PUT /api/v1/student/mock-exams/attempts/:id/items/:itemId/answer`, `POST /api/v1/student/mock-exams/attempts/:id/sections/:sectionId/submit`, `POST /api/v1/student/mock-exams/attempts/:id/submit`, `GET /api/v1/student/mock-exams/attempts/:id/review`, `GET /api/v1/student/mock-exams/history`
* **DB Table:** `assessment_templates`, `assessment_template_versions`, `assessment_modules`, `assessment_module_questions`, `assessment_attempts`, `assessment_attempt_modules`, `assessment_attempt_items`, `question_revisions`

---

### 3.3 Web App (`learning-web-app`)
Implements desktop and tablet mock exam workflows. The `/mock-exams` page displays available papers and completion statuses. Taking an exam opens a full-screen focused interface (`/mock-exams/[attemptId]`) featuring an `ExamTimer` countdown in the top bar, left-side question rail with answer/flag indicators, two-column split layout for passages and question stems, and section submit confirmation modals. After submission, the result screen offers comprehensive answer review and retake actions.

* **Code References:**
  * Mock exam catalog page: `src/app/(app)/mock-exams/page.tsx` (lines 20–105)
  * Full-screen exam page: `src/app/(focus)/mock-exams/[attemptId]/page.tsx` (lines 20–85)
  * Exam result page: `src/app/(app)/mock-exams/[attemptId]/result/page.tsx` (lines 20–110)
  * Timer component: `src/components/mock-exam/exam-timer.tsx` (lines 15–70)
  * Exam runner component: `src/components/mock-exam/exam-runner.tsx` (lines 35–205)
  * Exam rail navigator: `src/components/mock-exam/exam-navigator.tsx` (lines 15–75)
  * Review item component: `src/components/mock-exam/exam-review-item.tsx` (lines 15–60)
  * API service module: `src/lib/mock-exam/api.ts` (lines 20–110)
  * Query hooks: `src/lib/mock-exam/hooks.ts` (lines 20–85)
* **API:** `/api/student/mock-exams`, `/api/student/mock-exams/:id/attempts`, `/api/student/mock-exams/attempts/:id`, `/api/student/mock-exams/attempts/:id/items/:itemId/answer`, `/api/student/mock-exams/attempts/:id/submit`, `/api/student/mock-exams/attempts/:id/review`, `/api/student/mock-exams/history`
* **DB Table:** `assessment_templates`, `assessment_attempts`, `assessment_attempt_modules`, `assessment_attempt_items`

---

### 3.4 Admin Panel (`learning-app-admin`)
Provides an exam paper builder within the Admin Portal under `/exams`. Staff can view existing exams, create new drafts, configure overall versus section-based timing, select published questions via a searchable modal picker, arrange section sequences, and publish or unpublish papers. Live exams that have subsequent draft changes are highlighted with informative badges.

* **Code References:**
  * Exam list page: `src/features/exams/pages/exams-page.tsx` (lines 30–255)
  * Exam editor page: `src/features/exams/pages/exam-editor-page.tsx` (lines 35–305)
  * Duration field presets: `src/features/exams/components/duration-field.tsx` (lines 12–55)
  * Question picker modal: `src/features/exams/components/question-picker.tsx` (lines 25–140)
  * Section editor component: `src/features/exams/components/section-editor.tsx` (lines 20–110)
  * Exam API client: `src/features/exams/api/exams.api.ts` (lines 15–50)
  * Exam editor logic & validation: `src/features/exams/lib/exam-editor.ts` (lines 15–90)
* **API:** `GET /api/v1/admin/mock-exams`, `POST /api/v1/admin/mock-exams`, `GET /api/v1/admin/mock-exams/:id`, `PATCH /api/v1/admin/mock-exams/:id`, `POST /api/v1/admin/mock-exams/:id/publish`, `POST /api/v1/admin/mock-exams/:id/unpublish`, `DELETE /api/v1/admin/mock-exams/:id`
* **DB Table:** `assessment_templates`, `assessment_template_versions`, `assessment_modules`, `assessment_module_questions`

---

## 4. What Already Works (Zero Changes Needed)

1. **Robust revision snapshotting:** When a student begins a mock exam, durations and question revision pointers are locked into `assessment_attempt_modules` and `assessment_attempt_items`, insulating in-progress attempts from subsequent admin edits.
2. **Server-enforced section isolation:** In section-timed exams, questions in completed or locked sections cannot be edited or viewed, preventing cross-section time theft.
3. **No answer exposure before review:** API payloads completely strip `isCorrect` and explanations during execution, guaranteeing uncompromised test integrity.
4. **Resilient client timers:** Client countdowns calculate remaining time against absolute local deadlines derived from server timestamps (`deadlineAt`), preventing timer skew caused by sleeping tabs or app suspension.
5. **Idempotent submission:** Submitting a completed or expired exam multiple times returns the existing result without corrupting score counts or timestamps.
