# Timed Exams and Tests Developer Guide: Required Fields & Implementation

> **Target Audience:** Developers and AI Agents.  
> **Purpose:** Detailed reference for the Timed Exams and Practice Tests feature across Mobile App, Backend, Web App, and Admin Panel, covering server-owned clock enforcement, countdown mechanisms, timing modes (overall vs section-based), lazy expiration, network grace periods, and client resynchronization.

---

## 1. Required Fields & Originating Modules

Timing parameters can be established either by administrative staff (for structured Mock Exams) or by students (for self-directed Practice Tests).

### 1.1 Timing Configuration
*Captured in the Admin Exam Editor or Mobile/Web Practice Setup screens.*

| Field | Required | Where to Fill in App | Database Column | Notes |
|---|---|---|---|---|
| **Timing Mode** | Yes (Mock Exams) | Admin Exam Editor | `assessment_template_versions.timing_mode` | Either `'overall'` or `'per_section'`. |
| **Total Duration Minutes**| Conditional | Admin Exam Editor | `assessment_template_versions.total_duration_seconds` | Required when `timing_mode = 'overall'`. Stored in seconds (1 to 360 mins). |
| **Section Duration Minutes**| Conditional | Admin Section Editor | `assessment_modules.duration_seconds` | Required for each section when `timing_mode = 'per_section'`. |
| **Timed Toggle** | Optional (Practice) | Practice Setup | `assessment_attempts.configuration->>'timed'` | Boolean (`true` to enable countdown). |
| **Practice Duration** | Conditional | Practice Setup | `assessment_attempt_modules.duration_seconds` | Required if practice `timed = true` (5 to 180 mins). |

---

### 1.2 Runtime Clock & Expiration State
*Computed dynamically by the backend and tracked monotonically on clients.*

| Field | Required | Where to Fill in App | Database Column | Notes |
|---|---|---|---|---|
| **Started At** | Auto-set | Server at attempt start | `assessment_attempts.started_at`<br>`assessment_attempt_modules.started_at` | Timestamp marking clock initiation. |
| **Expires At** | Auto-calculated | Backend start handler | `assessment_attempts.expires_at`<br>`assessment_attempt_modules.expires_at` | Hard server deadline (`started_at + duration_seconds`). |
| **Remaining Seconds** | Auto-calculated | Dynamic API response | Calculated dynamically | `Math.max(0, ceil((expires_at - now) / 1000))`. `null` for untimed. |
| **Client Deadline** | Auto-derived | Client on response | `deadlineAt` (client memory) | Local epoch timestamp (`Date.now() + remainingSeconds * 1000`). |

---

## 2. Eligibility & Validation Rules Checklist

Before initiating or advancing a timed session, the system verifies:
1. **Server authority over time:** The client clock is purely a visual countdown. Start times, deadlines, and current timestamps are exclusively determined by the PostgreSQL database and NestJS server.
2. **Lazy settlement mechanism:** The system does not depend on background cron jobs. Every read or write operation calls `settle()`:
   * If `now >= expires_at` for an overall exam, the attempt is transitioned to `'expired'` and scored on existing answers.
   * If `now >= expires_at` for an open section, that section is marked `'completed'` (or `'expired'`), and the subsequent section is automatically started at the exact moment the previous section expired (preventing extra time on late return).
3. **5-Second network grace period:** In `acceptsAnswers()`, answer submissions are accepted up to `expires_at + 5 seconds` (`GRACE_SECONDS = 5`) to tolerate network transit delays without penalizing students.
4. **Section time isolation:** In section-timed exams, unspent time from an early section submission does NOT roll over into subsequent sections.
5. **Timed practice test bounds:** Practice test durations must fall between 5 and 180 minutes. When time expires, practice sessions transition to `'submitted'` status and compute final scores.
6. **Monotonic client countdown:** Clients convert server `remainingSeconds` to an absolute epoch deadline (`deadlineAt`). Countdown intervals measure against `deadlineAt - Date.now()`, ensuring the timer remains accurate if the laptop sleeps or the mobile app is paused in the background.
7. **Foreground clock resynchronization:** When the student returns to the mobile app (`AppState` becomes `'active'`) or focuses the web browser tab, the client triggers a background refetch to synchronize the clock with server truth.

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
│    Clock evaluated on every request via shared service │
│    POST /api/v1/student/practice/sessions (Timed)      │
│    POST /api/v1/student/mock-exams/:id/attempts        │
│    GET /api/v1/student/mock-exams/attempts/:id         │
│    PUT /api/v1/student/mock-exams/attempts/:id/.../ans │
│    POST /api/v1/student/mock-exams/attempts/:id/sec... │
└───────────────────────────┬────────────────────────────┘
                            │
                            ▼
┌────────────────────────────────────────────────────────┐
│           PostgreSQL Database (Neon DB)                │
│   tables: assessment_attempts (expires_at),            │
│   assessment_attempt_modules (duration_seconds,        │
│   expires_at), assessment_template_versions            │
└────────────────────────────────────────────────────────┘
```

### 3.1 Mobile App (`src/`)
Implements shared countdown timer logic across both practice tests and mock exams. `setup.tsx` includes a "Timed" switch with a duration selector (5 to 180 minutes) and recommended chips (~1.5 min per question). During testing, `exam-timer.tsx` is centered in the header, rendering formatted `mm:ss` text with color warnings (accent at normal, warning below 5 minutes, error below 1 minute). App backgrounding is handled in `use-exam-runner.ts` using `AppState` listeners to refetch remaining time upon resumption.

* **Code References:**
  * Header countdown component: `src/components/mock-exam/exam-timer.tsx` (lines 15–80)
  * Clock formatting & math utilities: `src/utils/exam-format.ts` (lines 10–55)
  * Timed practice setup screen: `src/app/practice/setup.tsx` (lines 75–125)
  * Timed practice runner screen: `src/app/practice/[sessionId].tsx` (lines 40–110)
  * Mock exam runner screen: `src/components/mock-exam/exam-runner.tsx` (lines 45–130)
  * AppState resync hook: `src/hooks/use-exam-runner.ts` (lines 35–85)
* **API:** `POST /student/practice/sessions` (`timed`, `durationMinutes`), `GET /student/mock-exams/attempts/:id`, `PUT /student/mock-exams/attempts/:id/items/:itemId/answer`
* **DB Table:** `assessment_attempts`, `assessment_attempt_modules`

---

### 3.2 Backend (`learning-app-backend`)
Provides the authoritative time engine in `src/shared/assessment/clock.ts`. Expiration dates are set at creation in `assessment_attempts.expires_at` and `assessment_attempt_modules.expires_at`. `MockExamsService.settle()` inspects deadlines before processing any read or write, closing expired sections or finalizing attempts. In `PracticeService`, timed practice sessions check `hasExpired()` on retrieval and auto-submit upon expiration.

* **Code References:**
  * Assessment clock library: `src/shared/assessment/clock.ts` (lines 1–28)
  * Mock exam clock & settlement service: `src/modules/mock-exams/mock-exams.service.ts` (lines 75–155, 230–310)
  * Practice timed session service: `src/modules/practice/practice.service.ts` (lines 90–140, 205–255)
  * Migration 0011 timing columns: `src/database/migrations/0011_mock-exams.sql` (lines 1–65)
  * Clock unit test suite: `test/unit/assessment-shuffle-clock.spec.ts` (lines 10–65)
* **API:** `POST /api/v1/student/practice/sessions`, `POST /api/v1/student/mock-exams/:id/attempts`, `GET /api/v1/student/mock-exams/attempts/:id`, `PUT /api/v1/student/mock-exams/attempts/:id/items/:itemId/answer`
* **DB Table:** `assessment_attempts`, `assessment_attempt_modules`, `assessment_template_versions`

---

### 3.3 Web App (`learning-web-app`)
Integrates the timing mechanism into the web desktop header via `ExamTimer`. The top navigation bar in `QuestionTopBar` mounts the countdown clock in its centered slot. `format.ts` provides `formatRemainingSeconds` and computes urgency classes. The timer component synchronizes with the server deadline on load and after every answer mutation. Upon reaching zero, the runner prompts the server to conclude the section or attempt.

* **Code References:**
  * Web countdown timer: `src/components/mock-exam/exam-timer.tsx` (lines 15–70)
  * Timer formatting helpers: `src/lib/mock-exam/format.ts` (lines 10–55)
  * Practice setup duration picker: `src/app/(app)/practice/setup/page.tsx` (lines 65–115)
  * Focus runner with timer slot: `src/components/practice/question-runner.tsx` (lines 40–110)
  * Top bar timer placement: `src/components/practice/question-topbar.tsx` (lines 15–45)
* **API:** `/api/student/practice/sessions`, `/api/student/mock-exams/attempts/:id`
* **DB Table:** `assessment_attempts`, `assessment_attempt_modules`

---

### 3.4 Admin Panel (`learning-app-admin`)
Allows administrators to define exam timing models and duration limits during paper creation. In `exam-editor-page.tsx`, staff toggle between single overall countdown and per-section timers. Preset duration buttons (30, 45, 60, 90, 120 minutes) and custom numerical inputs in `duration-field.tsx` write duration constraints into `assessment_template_versions` and `assessment_modules`.

* **Code References:**
  * Exam editor timing mode switch: `src/features/exams/pages/exam-editor-page.tsx` (lines 75–140)
  * Duration preset field: `src/features/exams/components/duration-field.tsx` (lines 10–55)
  * Section duration configuration: `src/features/exams/components/section-editor.tsx` (lines 35–85)
  * Exam editor payload serializer: `src/features/exams/lib/exam-editor.ts` (lines 30–80)
* **API:** `POST /api/v1/admin/mock-exams`, `PATCH /api/v1/admin/mock-exams/:id`
* **DB Table:** `assessment_template_versions`, `assessment_modules`

---

## 4. What Already Works (Zero Changes Needed)

1. **Immunity to client clock tampering:** Client-side system clock modifications have zero impact on server deadlines; deadlines are strictly evaluated against database `NOW()`.
2. **Deterministic section transition:** When a section deadline expires, the next section's `started_at` is tied to the previous section's `expires_at`, preventing students from gaining bonus time by intentionally staying offline.
3. **Grace period network cushion:** The 5-second `GRACE_SECONDS` buffer guarantees that valid answers submitted right as the timer hits zero are safely accepted.
4. **Drift-free tab suspension:** Web and mobile clients base countdown calculations on timestamp deltas (`deadlineAt - Date.now()`), remaining 100% accurate across device lock, sleep, or backgrounding.
5. **No cron dependency:** Attempt expiration is resolved lazily on access, preventing database locking and eliminating the need for periodic sweep workers.
