# Home Developer Guide: Required Fields & Implementation

> **Target Audience:** Developers and AI Agents.  
> **Purpose:** Detailed reference for the Student Home (Hub) feature across Mobile App, Backend, Web App, and Admin Panel, covering single-endpoint dashboard aggregation, timezone-aware streak tracking, countdown calculations, quick actions, code paths, APIs, and database schemas.

---

## 1. Required Fields & Originating Modules

The Student Home (Hub) screen consolidates student identity, target exam milestones, daily practice streaks, quick session entry points, and mock exam status into a single aggregated payload.

### 1.1 Home Dashboard Aggregation
*Delivered by `GET /api/v1/student/home` and presented on Mobile Hub and Web Home screens.*

| Field | Required | Where to Fill in App | Database Column | Notes |
|---|---|---|---|---|
| **Display Name** | Auto-derived | Onboarding / Auth profile | `users.full_name` or `users.email` | Preferred full name, falling back to email prefix if not provided. |
| **Streak Days** | Auto-calculated | Backend practice history | Computed from `assessment_attempts.submitted_at` | Number of consecutive calendar days with submitted attempts in user's timezone. |
| **Primary Exam Code**| Auto-derived | Signup / Onboarding | `exams.code` | `'sat'` or `'act'`. Derived from `user_exam_preferences` or goals. |
| **Exam Name** | Auto-derived | Database lookup | `exams.name` | Display name of the student's target test (e.g. `'SAT'`, `'ACT'`). |
| **Target Score** | Optional | Onboarding Step 3 | `student_exam_goals.target_score` | Target score threshold configured by the student. |
| **Exam Date** | Optional | Onboarding Step 3 | `student_exam_goals.exam_date` | Date string (`YYYY-MM-DD`). |
| **Days Remaining** | Auto-calculated | Server date arithmetic | Computed (`exam_date - today`) | Calendar days until exam date in student's timezone (`null` if unset/past). |
| **Active Session** | Optional | Resumed attempt check | `assessment_attempts` (`in_progress`) | Unfinished practice session metadata (`id`, `config`, progress). |
| **Recent Sessions** | Auto-derived | Recent attempts query | `assessment_attempts` (`submitted`) | Last 5 completed practice sessions with scores and timestamps. |
| **Bookmarked Count**| Auto-calculated | User bookmark query | Count from `question_bookmarks` | Total questions bookmarked by student for rapid review. |
| **Incorrect Count** | Auto-calculated | Mistakes retry query | Distinct count from `assessment_attempt_items` | Total distinct missed questions where latest answer was incorrect. |

---

## 2. Eligibility & Validation Rules Checklist

Before retrieving and constructing the Home dashboard, the system verifies:
1. **Student authorization:** The endpoint is secured by `StudentRoleGuard`. Parents and administrators receive 403.
2. **Onboarding completion guard:** Unfinished students are routed away from the Hub to `/resume` or `/onboarding` until `users.onboarded_at` is set.
3. **Single parallel aggregator query:** `HomeService.getHome` executes profile, exam goals, streak calculations, and practice summary queries concurrently via `Promise.all` to ensure sub-100ms response times without request waterfalls.
4. **Timezone-aware streak computation:**
   * Calculated against the student's registered timezone (`users.timezone`, defaulting to `'UTC'`).
   * Consecutive calendar days with at least one submitted session (`assessment_attempts.submitted_at`).
   * **Morning grace rule:** If the student has zero sessions today but completed a session yesterday, the streak remains intact, preventing the streak from prematurely dropping to 0 each morning.
   * Resets to 0 only when an entire calendar day has passed without practice.
   * Analyzes up to 400 days of historical activity.
5. **Days remaining calendar arithmetic:**
   * Computed as whole calendar days from midnight today to midnight on `student_exam_goals.exam_date` in the student's timezone.
   * Evaluates to `null` if no exam date is set or if the exam date has already passed.
6. **Dynamic mock exam card resolution:**
   * Highlights an in-progress mock exam first (for one-tap resume).
   * Highlights the first available unstarted mock exam second.
   * Displays `'coming_soon'` status if no mock exam papers are currently published.

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
│    GET /api/v1/student/home (Aggregated Hub Payload)   │
└───────────────────────────┬────────────────────────────┘
                            │
                            ▼
┌────────────────────────────────────────────────────────┐
│           PostgreSQL Database (Neon DB)                │
│   tables: users, user_exam_preferences, student_exam_  │
│   goals, exams, assessment_attempts, question_         │
│   bookmarks, assessment_attempt_items                  │
└────────────────────────────────────────────────────────┘
```

### 3.1 Mobile App (`src/`)
Serves as the primary student entry screen (`(tabs)/hub.tsx`). Displays a personalized welcome banner with the student's display name, daily streak badge, target exam focus card with days remaining countdown (`focus-card.tsx`), Quick Access buttons (20-question practice, bookmarked, incorrect retry), a status-aware mock exam highlight card (`mock-exam-card.tsx`), and recent session history. Supports pull-to-refresh and automatically refetches when the tab regains focus.

* **Code References:**
  * Hub screen: `src/app/(tabs)/hub.tsx` (lines 35–225)
  * Target exam focus card: `src/components/hub/focus-card.tsx` (lines 15–60)
  * Mock exam highlight card: `src/components/home/mock-exam-card.tsx` (lines 15–70)
  * Section header component: `src/components/home/section-header.tsx` (lines 10–35)
  * Home API client: `src/services/home-api.ts` (lines 10–28)
  * Home query hook: `src/hooks/use-home.ts` (lines 12–35)
  * Quick session hook: `src/hooks/use-quick-session.ts` (lines 10–35)
* **API:** `GET /student/home`
* **DB Table:** `users`, `student_exam_goals`, `user_exam_preferences`, `exams`, `assessment_attempts`, `question_bookmarks`

---

### 3.2 Backend (`learning-app-backend`)
Provides the aggregation layer in `HomeModule`. `HomeController.getHome` delegates to `HomeService.getHome`, which reads the student's profile, primary exam, target score, test date, and attempt timestamps, and calls `PracticeService.summary()` to assemble the complete dashboard payload in a single database round-trip. `home.logic.ts` encapsulates pure algorithmic calculations for `computeStreak` and `daysRemaining`.

* **Code References:**
  * Home controller: `src/modules/home/home.controller.ts` (lines 20–50)
  * Home service aggregator: `src/modules/home/home.service.ts` (lines 15–35)
  * Streak & days remaining logic: `src/modules/home/home.logic.ts` (lines 1–50)
  * Home database repository: `src/modules/home/repositories/home.repository.ts` (lines 20–105)
  * Practice summary reuse: `src/modules/practice/practice.service.ts` (lines 280–320)
  * Home unit test suite: `test/unit/home.spec.ts` (lines 10–65)
* **API:** `GET /api/v1/student/home`
* **DB Table:** `users`, `student_exam_goals`, `user_exam_preferences`, `exams`, `assessment_attempts`, `question_bookmarks`, `assessment_attempt_items`

---

### 3.3 Web App (`learning-web-app`)
Implements the responsive desktop student dashboard at `/hub`. Renders the welcome greeting, streak pill, countdown focus card (`focus-card.tsx`), quick practice launch controls, mock exam card, and recent study activity. The dashboard proxies through the Next.js BFF route `/api/student/home` and manages cache invalidation via TanStack Query.

* **Code References:**
  * Web hub page: `src/app/(app)/hub/page.tsx` (lines 20–95)
  * Focus card banner: `src/components/home/focus-card.tsx` (lines 15–50)
  * Mock exam card widget: `src/components/home/mock-exam-card.tsx` (lines 15–55)
  * Client API fetcher: `src/lib/home/api.ts` (lines 10–25)
  * React Query hook: `src/lib/home/hooks.ts` (lines 10–30)
* **API:** `/api/student/home` (proxied to backend)
* **DB Table:** `users`, `student_exam_goals`, `exams`, `assessment_attempts`

---

### 3.4 Admin Panel (`learning-app-admin`)
The admin panel does not render a student home screen; instead, it maintains the underlying taxonomies, active exam configurations (`exams` table), and published mock exams that supply the student Home view with live targets, countdowns, and exam paper highlights.

* **Code References:**
  * Taxonomy subject catalog: `src/features/content/pages/subjects-page.tsx` (lines 25–95)
  * Mock exam papers management: `src/features/exams/pages/exams-page.tsx` (lines 30–255)
* **API:** `GET /api/v1/admin/taxonomy`, `GET /api/v1/admin/mock-exams`
* **DB Table:** `taxonomy_nodes`, `exams`, `assessment_templates`

---

## 4. What Already Works (Zero Changes Needed)

1. **Zero waterfall dashboard loading:** One single endpoint (`GET /student/home`) returns student name, streak, target test, countdown, active test, and recent scores, eliminating multi-request latency.
2. **Timezone-accurate streaks:** Streak calculation runs using the user's explicit local timezone, ensuring midnight rollovers correspond to the student's actual geographical day.
3. **Morning streak protection:** Students waking up to a new day do not see their streak reset to zero; yesterday's completed practice holds the streak active until midnight of the current day.
4. **Dynamic exam countdowns:** Target test days remaining automatically update daily based on server calendar differences, automatically vanishing once the test date passes.
5. **Shared cache invalidation:** Submitting any practice test or toggling a bookmark automatically invalidates and refreshes the Home query cache across the app.
