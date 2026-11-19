# Plan: Onboarding, End to End (Mobile + Backend + Database)

Date: 2026-11-19
Apps: Mobile app (`D:\lm`) and Backend (`learning-app-backend`)

## Status (2026-11-19): implemented

- Done: migrations 0008 and 0009, the `student` backend module, integration tests (32 new, all pass), the mobile changes, `architecture.md` section 10d.
- Differences from this plan: the finished-onboarding column is `users.onboarded_at` (not `onboarding_completed_at`); the seed file now sets the exam score range; the exam list shows the exam name in front of a subject label (for example "SAT Mathematics") when a student has both exams.
- Not yet done: migrations are applied to the local test database only, not to the Neon dev database. The manual end-to-end checklist in section 9 still needs a run on a real device.

## 1. Goal

Make the onboarding flow work for real, from the phone to the database and back:

- The student answers 6 steps in the app.
- Each answer is saved in the database through the backend API.
- When the student comes back (re-login, new phone), the app loads the saved answers and resumes at the right step.

Right now the backend has **no onboarding API and no onboarding tables**. In dev, the app pretends it works by using a fake in-memory mock. This plan replaces the mock with the real thing.

## 2. Decisions already made

| # | Decision |
|---|---|
| 1 | Remove **Video lessons** from "How do you like to study?". Skill Level screen: no change. |
| 2 | Subjects come **only from the database**. Nothing hardcoded in the app. |
| 3 | Remove **AP Exams** completely from the mobile app. Only SAT and ACT exist. |
| 4 | Signup choice **"Other – A different test" is hidden in the mobile app with a flag** (`SHOW_OTHER_TEST = false`). Backend and DB still accept `other`, so it can be enabled later. Existing "other" users see SAT and ACT subjects, since AP is gone. |
| 5 | ACT **Writing** stays in the subject list (it is active in the DB). |

## 3. What exists today

**Mobile app (done, but uses a mock)**
- Screens in `src/app/(onboarding)/`: subjects, academic, goals, skill-levels, study-preferences, learning-goals, resume.
- `src/services/onboarding-api.ts` already calls the API routes listed in section 6.
- `src/services/onboarding-mock.ts` takes over when the API says 404 (dev only). It holds the hardcoded subjects, AP and score ranges.
- After login, `use-after-auth.ts` sends students with `onboardingCompleted = false` to the `/resume` screen.

**Backend (not done)**
- No `/student/*` routes. Only `auth`, `admin/auth`, `health`, `admin/questions`, `admin/passages`, `admin` (taxonomy).
- `users.onboarding_completed_at` exists, but nothing sets it. So a real student would be sent to onboarding on every login.

**Database (partly ready)**
- `exams`: `sat`, `act` (the type `exam_code` allows only these two).
- `taxonomy_nodes`: subjects are rows with `node_type = 'section'` and `parent_id IS NULL`.
  - SAT: Reading and Writing, Mathematics.
  - ACT: English, Mathematics, Reading, Science, Writing.
  - Admins can add, rename, reorder, archive (`is_active = false`). So the DB is the right source.
- `users.target_test` has `sat | act | both | other` (set at signup).
- The subject `code` repeats across exams (`mathematics` is in both SAT and ACT), so the app must use the subject **id (UUID)**, never the code.

---

## 4. Part A: Small changes in the mobile app

### A1. Remove "Video lessons"
- `src/data/onboarding-options.ts`: delete the `video_lessons` line from `STYLES`.
- `src/services/onboarding-api.ts`: remove `'video_lessons'` from `LearningStyle`.
- Profile screen uses the same list, so it is fixed automatically.
- Allowed styles from now on: `practice_questions`, `written_explanations`, `interactive_learning`.

### A2. Remove AP
- `onboarding-api.ts`: `OnboardingExam = 'sat' | 'act'`.
- `goals.tsx`: remove `ap` from the starting drafts and from the comment about AP scores.
- `onboarding-mock.ts` is deleted in Part D.

### A3. Hide "Other" at signup with a flag (mobile only)
- `src/app/(auth)/signup-test.tsx`: add a flag `SHOW_OTHER_TEST = false` next to the `TESTS` list. When it is `false`, the "Other" option is not shown. To bring it back later, set the flag to `true`.
- **Backend and DB do not change.** The auth API, the `users.target_test` check (`sat | act | both | other`) and `saveExamPreferences` stay exactly as they are. No signup migration is needed.
- The `TargetTest` type in the app keeps `'other'`, because the API can still return it for existing accounts.
- Existing users who already chose "Other" keep working (see below).
- New exams later: add the exam to the `exam_code` type, `exams` table (with score range) and `taxonomy_nodes`, update the "How `exams` is built" rule, and show the new option in the app. Subjects and score ranges come from the DB, so the onboarding screens need no new hardcoded lists. "Other" is only a placeholder for "a test we do not cover yet".
- What "Other" shows in onboarding (for existing users): Before, it showed AP. AP is removed, and we only cover SAT and ACT, so a student who chose **Other** sees **both SAT and ACT** subjects (same as "Both" and as students with an empty test choice). See "How `exams` is built" in section 6.
- `saveExamPreferences` in `users.repository.ts` keeps its current behavior for `'other'` (it stores nothing in `user_exam_preferences`).

---

## 5. Part B: Database (new migrations)

Follow `skills.md`: every migration has an up and a down part, explicit constraints, no `SELECT *`.

### Migration 0008: exam score range
Add to `exams`: `score_min`, `score_max`, `score_step` (smallint, not null).
- SAT: 400 / 1600 / 10
- ACT: 1 / 36 / 1

The app uses these for the target score hints and validation, so nothing is hardcoded in the app.

### Migration 0009: onboarding tables (4 tables)
All tables are keyed by `user_id` with `ON DELETE CASCADE` from `users`.

| Table | Columns (main) | Notes |
|---|---|---|
| `student_profiles` | `user_id` PK (one row per student), `current_step` (1–6, default 1), `date_of_birth` date null, `country` text null, `grade` text null, `graduation_year` smallint null, `school_name` varchar(120) null, `preparation_goal` varchar(300) null, `daily_minutes` smallint null, `preferred_times` text[] (default empty), `study_days` text null, `learning_styles` text[] (default empty), `created_at`, `updated_at` | Holds the step, academic details (step 2), preparation goal (step 3) and study preferences (step 5). |
| `student_subjects` | `user_id`, `subject_id` → `taxonomy_nodes(id)` ON DELETE CASCADE, `skill_level` text not null default `'not_sure'` | PK (`user_id`, `subject_id`). Subjects are saved in step 1, `skill_level` in step 4. |
| `student_exam_goals` | `user_id`, `exam` exam_code, `target_score` smallint null, `exam_date` date null | PK (`user_id`, `exam`). Step 3. |
| `student_learning_goals` | `user_id`, `goal` text | PK (`user_id`, `goal`). 6 allowed values. Step 6. |

Notes on this structure:
- **No JSONB.** Study preferences are normal columns and arrays, so check constraints can reject bad values.
- **Profile columns allow NULL**, because the row is created at step 1 before the academic details exist. The service checks that required values are present before "complete".
- **Check constraints:** `current_step` 1–6; `grade` (7 allowed values); `daily_minutes` in 15/30/60/120; `study_days` in weekdays/weekends/every_day; `preferred_times` only morning/afternoon/evening/night; `learning_styles` only the 3 allowed styles; `skill_level` in beginner/intermediate/advanced/not_sure; `goal` in the 6 allowed values.
- **Profile row creation:** it is created on the first GET or first save.
- Completion uses the existing `users.onboarding_completed_at` (single source of truth for "finished").

Seed data: confirm the dev database has the SAT/ACT subjects from `seeds/0001_exams-and-taxonomy.sql` (otherwise the subject list will be empty).

---

## 6. Part C: Backend API

New module: `src/modules/student/` (copy the shape of the `auth` module): controller, service, repository, `schemas/` (Zod), `interfaces/`.

All routes:
- Need a logged-in **student** (`AccessTokenGuard`). Parents and admins get 403.
- User id comes from the token, never from the request body.
- Use `z.strictObject` for bodies and a rate limit (as other routes do).
- Return the same response envelope as the auth routes.

| Route | Step | Body | Result |
|---|---|---|---|
| `GET /v1/student/onboarding` | | | Full state |
| `PUT /v1/student/onboarding/subjects` | 1 | `{ subjects: string[] }` (subject ids) | Full state |
| `PUT /v1/student/onboarding/academic` | 2 | `{ dateOfBirth, country, grade, graduationYear, schoolName }` | Full state |
| `PUT /v1/student/onboarding/goals` | 3 | `{ goals: [{exam, targetScore, examDate}], preparationGoal }` | Full state |
| `PUT /v1/student/onboarding/skill-levels` | 4 | `{ levels: { [subjectId]: level } }` | Full state |
| `PUT /v1/student/onboarding/study-preferences` | 5 | `{ dailyMinutes, preferredTimes, studyDays, learningStyles }` | Full state |
| `POST /v1/student/onboarding/complete` | 6 | `{ learningGoals: string[] }` | Full state |
| `GET /v1/student/profile` | later edits | | Full state (same as GET onboarding) |

### The "full state" response (what the app already expects, with ids and score range added)
```
{
  onboarding: { currentStep: 1..6, completed: boolean },
  exams: [
    { exam: "sat", name: "SAT",
      scoreRange: { min: 400, max: 1600, step: 10 },
      subjects: [ { id: "<uuid>", name: "Mathematics" }, ... ] }
  ],
  subjects: [ "<subject id>", ... ],
  profile: { dateOfBirth, country, grade, graduationYear, schoolName } | null,
  goals: [ { exam, targetScore, examDate } ],
  preparationGoal: string | null,
  skillLevels: { "<subject id>": "beginner" ... },
  studyPreferences: { ... } | null,
  learningGoals: [ ... ]
}
```
The app's `ExamInfo.subjects` changes from `{code, name}` to `{id, name}`.

### How `exams` is built
1. Read the student's `target_test`: `sat` → [SAT], `act` → [ACT], `both`, `other` or `NULL` → [SAT, ACT].
2. For each exam, read its **active** top-level subjects from `taxonomy_nodes`, ordered by `position`.
3. Skip an exam if it has no active subjects.

### Business rules (service layer)
- **Subjects:** at least 1; every id must be an active subject of an allowed exam, else 422. Replace the saved list in one transaction. Subjects that stay keep their `skill_level`; new ones start as `not_sure`; removed ones are deleted together with their level. If an exam has no subjects left, delete that exam's goal.
- **Academic:** age between 5 and 100; grade in the allowed list; graduation year from last year to 7 years ahead; school name max 120.
- **Goals:** only exams that have a selected subject; score inside the exam's `score_min`–`score_max` (read from DB); exam date not in the past and within 3 years; preparation goal max 300. Everything is optional. An empty save is allowed (that is "Skip").
- **Skill levels:** keys must be the student's selected subjects. The level is updated on the existing `student_subjects` rows (no separate table).
- **Study preferences:** only allowed values. All optional (Skip allowed).
- **Complete:** needs steps 1–5 done (`current_step` is 6) and at least one subject. Saves learning goals (at least 1), sets `users.onboarding_completed_at = now()` once, and keeps the first value if called again (safe to retry).
- **Progress:** saving step N sets `current_step = max(current_step, N + 1)`, capped at 6. A student can re-save an earlier step. Saving a step that is too far ahead is rejected.
- **Editing later (Profile → Edit):** the same PUT routes work after completion. `completed` stays true.
- Each save runs in one DB transaction and returns the full state, so the app does not need to refetch.

### Tests
- Unit tests for the rules above (valid and invalid cases).
- Integration tests: full happy path 1→6, resume after "restart", wrong role (parent/admin) gets 403, no token gets 401, archived subject is hidden and rejected, `NULL` target test shows both exams, repeat `complete` is safe.
- Run: `npm run typecheck`, `npm run lint`, `npm test`, `npm run test:integration`, and migrate up → down → up.
- Update `architecture.md` (new module, tables, routes; remove "onboarding is planned").

---

## 7. Part D: Mobile app, connect to the real API

1. `onboarding-api.ts`
   - Subjects are `{ id, name }`. Saved and loaded by id.
   - Remove `withMock` and the `useMock` flag. Call the real API only.
2. **Delete** `src/services/onboarding-mock.ts`.
3. `subjects.tsx`: `subjectEmoji` guessed from the code (`sat_math`). Codes are now `mathematics`, `reading-and-writing`, etc. Update it to match these, or use one neutral icon. Use the id as the list key and for selection.
4. `goals.tsx`: use `scoreRange` from the API (already does). Remove AP. Check the "suggested scores" helper still works.
5. `skill-levels.tsx` and `onboarding-details.tsx` (Profile): look up subject names by id.
6. `use-onboarding.ts`: no logic change expected. Keep `staleTime: Infinity` and the cache update after each save.
7. Error handling: if loading fails, show the message and a "Try again" button (the loader and Profile already have one; confirm the step screens do).
8. Account switch: `useAfterAuth` already clears `['student']` cache on login. Keep it.

---

## 8. Order of work

1. Part B: migrations 0008, 0009 (and run them on dev DB).
2. Part C: backend module, routes, tests. Test with curl or Swagger before touching the app.
3. Part A + D: mobile changes (video, AP, hide Other with flag, real API, delete mock).
4. End-to-end test (section 9).
5. Update docs (`architecture.md`).

## 9. End-to-end test checklist

- [ ] Sign up as a new student with SAT only → Subjects screen shows SAT subjects only, from the DB.
- [ ] Sign up with "Both" → SAT and ACT subjects, ACT Writing included.
- [ ] Signup shows only SAT, ACT and Both (no "Other"). Setting the flag to `true` shows it again.
- [ ] An existing account with `target_test = 'other'` → onboarding shows SAT and ACT subjects (no AP).
- [ ] Archive a subject in the admin panel → it disappears from the app after reload.
- [ ] Complete steps 1–6 → rows exist in all 4 tables (`student_profiles`, `student_subjects` with skill levels, `student_exam_goals`, `student_learning_goals`) → `users.onboarding_completed_at` is set.
- [ ] Study screen has 3 styles, no Video lessons (also not in Profile).
- [ ] Goals screen has no AP.
- [ ] Close the app in the middle of step 3 → log in again → app resumes at step 3 with saved answers.
- [ ] Finish onboarding → log out → log in → goes straight to the dashboard (not onboarding).
- [ ] Profile → Edit a section → change saved in DB → Profile shows the new value.
- [ ] Log in as a parent and as an admin → onboarding API returns 403, no onboarding screen.
- [ ] Turn the server off → clear error message and "Try again", no fake data.
- [ ] No file named `onboarding-mock` and no text "AP" or "video_lessons" left in the app.

## 10. Risks and notes

- **Subject deleted by an admin:** `ON DELETE CASCADE` removes it from students' saved answers. Archiving (the normal way) keeps old answers but hides the subject from new choices.
- **Users with "other":** they see both SAT and ACT. This is the same as for "Both", so it is safe. The API can still return `other`, so the app type must keep it.
- **Google login:** Google is on the login screen only. The backend (`googleLogin`) signs in an existing account and returns "please sign up first" for an unknown email. So every Google user has already done the normal email signup, including the test question, and has `target_test` filled in. No special rule is needed. An empty `target_test` (shown as both exams) only happens for old accounts created before the question existed. If "Sign up with Google" is added later, it must ask the test question first (separate task, not in this plan).
- **Subject ids in the app cache:** old cached state from the mock used codes like `sat_math`. The mock was never stored on the server, so nothing to migrate. A fresh app start clears it.
- **Parents:** the parent role also picked a test at signup. Onboarding is for students only, so parents are unaffected.
