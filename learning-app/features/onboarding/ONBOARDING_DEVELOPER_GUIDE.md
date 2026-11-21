# Onboarding Developer Guide: Required Fields & Implementation

> **Target Audience:** Developers and AI Agents.  
> **Purpose:** Detailed reference for the Student Onboarding feature across Mobile App, Backend, Web App, and Admin Panel, covering multi-step data collection, validation rules, code paths, APIs, and database schemas.

---

## 1. Required Fields & Originating Modules

Student Onboarding is a 6-step guided workflow that personalizes study plans and baseline skill levels for registered students.

### 1.1 Step 1: Subject Selection
*Captured in Step 1 of onboarding (`/subjects`).*

| Field | Required | Where to Fill in App | Database Column | Notes |
|---|---|---|---|---|
| **Subject IDs** | Yes (at least 1) | Subjects screen | `student_subjects.subject_id` | UUID array referencing active `taxonomy_nodes.id`. |
| **Exam Code** | Auto-derived | Inferred from subject | `student_subjects.exam` | Matches the student's chosen exam (`'sat'`, `'act'`). |

---

### 1.2 Step 2: Academic Details
*Captured in Step 2 of onboarding (`/academic`).*

| Field | Required | Where to Fill in App | Database Column | Notes |
|---|---|---|---|---|
| **Date of Birth** | Yes | Academic screen | `student_profiles.date_of_birth` | ISO date (`YYYY-MM-DD`). Must represent age 5 to 100. |
| **Country** | Yes | Academic screen | `student_profiles.country` | ISO 3166-1 alpha-2 country code (for example, `'US'`). |
| **Grade** | Yes | Academic screen | `student_profiles.grade` | Allowed: `grade_9`, `grade_10`, `grade_11`, `grade_12`, `college_freshman`, `college_sophomore`, `other`. |
| **Graduation Year** | Yes | Academic screen | `student_profiles.graduation_year` | 4-digit integer within the valid future/past threshold. |
| **School Name** | Optional | Academic screen | `student_profiles.school_name` | Free-text string (up to 200 characters). |

---

### 1.3 Step 3: Exam Goals & Preparation
*Captured in Step 3 of onboarding (`/goals`).*

| Field | Required | Where to Fill in App | Database Column | Notes |
|---|---|---|---|---|
| **Target Score** | Optional | Goals screen | `student_exam_goals.target_score` | Must be between `exams.score_min` and `score_max` and match `score_step`. |
| **Exam Date** | Optional | Goals screen | `student_exam_goals.exam_date` | Must be a future date (up to 3 years ahead). |
| **Preparation Goal** | Optional | Goals screen | `student_profiles.preparation_goal` | Allowed: `top_percentile`, `score_improvement`, `concept_mastery`, `practice_only`. |

---

### 1.4 Step 4: Subject Skill Levels
*Captured in Step 4 of onboarding (`/skill-levels`).*

| Field | Required | Where to Fill in App | Database Column | Notes |
|---|---|---|---|---|
| **Skill Level** | Yes (per subject) | Skill Levels screen | `student_subjects.skill_level` | Allowed: `beginner`, `intermediate`, `advanced`, `not_sure`. |

---

### 1.5 Step 5: Study Preferences
*Captured in Step 5 of onboarding (`/study-preferences`).*

| Field | Required | Where to Fill in App | Database Column | Notes |
|---|---|---|---|---|
| **Daily Minutes** | Optional | Study Preferences screen | `student_profiles.daily_minutes` | Allowed: `15`, `30`, `60`, or `120`. |
| **Preferred Times** | Optional | Study Preferences screen | `student_profiles.preferred_times` | Array of: `morning`, `afternoon`, `evening`, `night`. |
| **Study Days** | Optional | Study Preferences screen | `student_profiles.study_days` | Allowed: `weekdays`, `weekends`, `every_day`. |
| **Learning Styles** | Optional | Study Preferences screen | `student_profiles.learning_styles` | Array of: `practice_questions`, `written_explanations`, `interactive_learning`. |

---

### 1.6 Step 6: Learning Goals & Completion
*Captured in Step 6 of onboarding (`/learning-goals`).*

| Field | Required | Where to Fill in App | Database Column | Notes |
|---|---|---|---|---|
| **Learning Goals** | Yes (at least 1) | Learning Goals screen | `student_learning_goals.goal` | Array of allowed goals (for example, `test_strategies`, `weak_areas`). |
| **Onboarded Timestamp** | Auto-set | Server completion trigger | `users.onboarded_at` | Set to `NOW()` upon successful validation of all 6 steps. |

---

## 2. Eligibility & Validation Rules Checklist

Before updating or completing onboarding, the system verifies:
1. **Student role requirement:** The caller must hold the `student` role (`StudentRoleGuard`). Parent and admin accounts cannot onboard through these routes.
2. **Subject validity:** In step 1, every submitted `subjectId` must belong to an active taxonomy node associated with the student's registered test (`INVALID_SUBJECT`).
3. **Sequential progression:** Steps must be completed in order. `StudentOnboardingService.saveStep` verifies that `profile.currentStep` has reached or passed the step being submitted.
4. **Age boundary:** Date of birth must place the student between 5 and 100 years old.
5. **Score boundaries:** Target scores in step 3 must fit within the configured `score_min` and `score_max` defined in the `exams` table, aligned with `score_step` increments.
6. **Target date validity:** Exam target dates must fall between the current date and three years in the future.
7. **Subject skill consistency:** Skill levels in step 4 must only be submitted for subjects selected in step 1.
8. **Prerequisites on completion:** `POST /student/onboarding/complete` verifies that steps 1 through 5 are complete and at least one subject is selected. Incomplete submissions return 409 (`ONBOARDING_INCOMPLETE`).
9. **Idempotent completion:** Calling `complete` on an already-onboarded profile is safe and leaves the original `users.onboarded_at` timestamp unchanged.

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
│    GET /api/v1/student/onboarding (Resume / State)     │
│    PUT /api/v1/student/onboarding/subjects (Step 1)    │
│    PUT /api/v1/student/onboarding/academic (Step 2)    │
│    PUT /api/v1/student/onboarding/goals (Step 3)       │
│    PUT /api/v1/student/onboarding/skill-levels (Step 4)│
│    PUT /api/v1/student/onboarding/study-preferences (5)│
│    POST /api/v1/student/onboarding/complete (Step 6)   │
└───────────────────────────┬────────────────────────────┘
                            │
                            ▼
┌────────────────────────────────────────────────────────┐
│           PostgreSQL Database (Neon DB)                │
│   tables: student_profiles, student_subjects,          │
│           student_exam_goals, student_learning_goals,  │
│           users, taxonomy_nodes, exams                 │
└────────────────────────────────────────────────────────┘
```

### 3.1 Mobile App (`src/`)
Features a 6-step interactive screen sequence and resume coordinator that guides students through configuring study preferences, target test scores, and baseline subject skill levels. When returning, `resume.tsx` queries `GET /api/v1/student/onboarding` and routes the student directly to their earliest incomplete step (`currentStep`, 1 to 6). Each step validates its form locally and calls the corresponding `PUT /student/onboarding/*` endpoint. Step 6 calls `POST /student/onboarding/complete`, shows `welcome.tsx`, and navigates to the main study hub (`/hub`).

* **Code References:**
  * Resume coordinator: `src/app/(onboarding)/resume.tsx` (lines 10–19)
  * Step 1 Subjects screen: `src/app/(onboarding)/subjects.tsx` (lines 31–71)
  * Step 2 Academic screen: `src/app/(onboarding)/academic.tsx` (lines 33–70)
  * Step 3 Goals screen: `src/app/(onboarding)/goals.tsx` (lines 37–76)
  * Step 4 Skill Levels screen: `src/app/(onboarding)/skill-levels.tsx` (lines 21–35)
  * Step 5 Study Preferences screen: `src/app/(onboarding)/study-preferences.tsx` (lines 28–54)
  * Step 6 Learning Goals screen: `src/app/(onboarding)/learning-goals.tsx` (lines 20–47)
  * Welcome screen: `src/app/(onboarding)/welcome.tsx` (lines 14–40)
  * Onboarding API client: `src/services/onboarding-api.ts` (lines 88–100)
  * Query hook: `src/hooks/use-onboarding.ts` (lines 15–40)
* **API:** `GET /api/v1/student/onboarding`, `PUT /api/v1/student/onboarding/subjects`, `academic`, `goals`, `skill-levels`, `study-preferences`, `POST /api/v1/student/onboarding/complete`
* **DB Table:** `student_profiles`, `student_subjects`, `student_exam_goals`, `student_learning_goals`, `users`, `taxonomy_nodes`, `exams`

---

### 3.2 Backend (`learning-app-backend`)
Provides a transactional onboarding service managing multi-step progression, step validation, and completion flags while securing endpoints under `StudentRoleGuard`. `StudentOnboardingService.getState` retrieves the student's exam, available taxonomy subjects, saved responses, and computes `currentStep`. Each `PUT` endpoint validates request payloads with Zod and updates the database inside a transaction. `complete` verifies that all previous steps are satisfied, updates `users.onboarded_at = NOW()`, and returns the finalized state.

* **Code References:**
  * Student controller: `src/modules/student/student.controller.ts` (lines 57–266)
  * Onboarding state retrieval: `src/modules/student/student-onboarding.service.ts` (lines 114–191)
  * Step save handlers: `src/modules/student/student-onboarding.service.ts` (lines 193–320)
  * Completion service handler: `src/modules/student/student-onboarding.service.ts` (lines 322–344)
  * Atomic step save helper: `src/modules/student/student-onboarding.service.ts` (lines 350–373)
  * Onboarding repository: `src/modules/student/repositories/student-onboarding.repository.ts` (lines 47–220)
* **API:** `GET /api/v1/student/onboarding`, `PUT /api/v1/student/onboarding/subjects`, `academic`, `goals`, `skill-levels`, `study-preferences`, `POST /api/v1/student/onboarding/complete`
* **DB Table:** `student_profiles`, `student_subjects`, `student_exam_goals`, `student_learning_goals`, `users`, `taxonomy_nodes`, `exams`

---

### 3.3 Web App (`learning-web-app`)
Implements a responsive Next.js App Router multi-step onboarding flow matching the mobile questionnaire. Pages under `src/app/(onboarding)/onboarding/` fetch state through `src/lib/onboarding/api.ts` via the Next.js BFF proxy. Each step posts progress to `/api/student/onboarding/*`. When step 6 completes, the student is redirected to the web student dashboard.

* **Code References:**
  * Onboarding entry index: `src/app/(onboarding)/onboarding/page.tsx` (lines 14–26)
  * Step 1 Subjects page: `src/app/(onboarding)/onboarding/subjects/page.tsx` (lines 24–52)
  * Step 2 Academic page: `src/app/(onboarding)/onboarding/academic/page.tsx` (lines 32–73)
  * Step 3 Goals page: `src/app/(onboarding)/onboarding/goals/page.tsx` (lines 36–76)
  * Step 4 Skill Levels page: `src/app/(onboarding)/onboarding/skill-levels/page.tsx` (lines 20–38)
  * Step 5 Study Preferences page: `src/app/(onboarding)/onboarding/study-preferences/page.tsx` (lines 28–54)
  * Step 6 Learning Goals page: `src/app/(onboarding)/onboarding/learning-goals/page.tsx` (lines 20–46)
  * Client API functions: `src/lib/onboarding/api.ts` (lines 16–28)
* **API:** `GET /api/student/onboarding`, `PUT /api/student/onboarding/*`, `POST /api/student/onboarding/complete`
* **DB Table:** `student_profiles`, `student_subjects`, `student_exam_goals`, `student_learning_goals`, `users`, `taxonomy_nodes`, `exams`

---

### 3.4 Admin Panel (`learning-app-admin`)
Provides a content management interface for subjects, domains, and topics that directly populate step 1 of student onboarding without requiring backend code deployments. Staff access the Subjects catalog under Content (`subjects-page.tsx`), adding or modifying taxonomy nodes with `node-form-dialog.tsx`. All changes are stored in `taxonomy_nodes` and are immediately served to onboarding students.

* **Code References:**
  * Subjects management page: `src/features/content/pages/subjects-page.tsx` (lines 25–95)
  * Taxonomy node dialog: `src/features/content/components/node-form-dialog.tsx` (lines 30–85)
  * Content API service: `src/features/content/api/content.api.ts` (lines 29–82)
* **API:** `GET /api/v1/admin/content/taxonomy`, `POST /api/v1/admin/content/nodes`, `PUT /api/v1/admin/content/nodes/:id`
* **DB Table:** `taxonomy_nodes`

---

## 4. What Already Works (Zero Changes Needed)

1. **Automatic resume workflow:** Both mobile and web frontends dynamically resume unfinished students at their earliest incomplete step.
2. **Partial step persistence:** Each step saves independently to PostgreSQL; users who leave mid-flow never lose completed answers.
3. **Dynamic taxonomy discovery:** Subject selection pulls live nodes from `taxonomy_nodes` filtered by the student's selected exam.
4. **Validation against exam thresholds:** Target scores are checked against min, max, and step increments defined in the `exams` table.
5. **Atomic onboarding finalization:** Step 6 verifies all required steps are satisfied before timestamping `users.onboarded_at`.
