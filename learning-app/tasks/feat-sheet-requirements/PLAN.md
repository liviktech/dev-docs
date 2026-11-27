# Feature sheet requirements: execution plan

Backend requirements from the feature sheet, worked one point at a time. Each point is evaluated against the live code across all four repositories in the project.

This revision covers only the points already planned (P1-008, P1-006, P1-010, P1-011, P1-012, P1-013, P1-014, P1-015, P1-016). The remaining points will be added later.

### Target Repositories
1. **Backend**: `D:\Livik-projects\Learning-app\learning-app-backend`
2. **Admin Web**: `D:\Livik-projects\Learning-app\learning-app-admin`
3. **Student/Parent Web**: `D:\Livik-projects\Learning-app\learning-web-app`
4. **Mobile App**: `D:\lm`

### Ground rules for every point
- **Additive and non-breaking.** Use expand/contract: add new structures, dual-write if needed, switch reads, then remove the old structure in a later migration. Old mobile builds must keep working until the contract step.
- **Migration numbers are assigned at implementation time** (the latest today is `0017`). The numbers below are placeholders in order.
- **One source of truth** for every fact. No column is kept in two tables unless one is explicitly marked as a derived copy.
- **Docs and tests ship with the code.** Every point updates `architecture.md` (project rule) and adds unit and integration tests in the existing `test/unit` and `test/integration` layout.
- **Swagger.** Every changed request or response schema updates the Zod response schemas so OpenAPI stays correct.

### Implementation order (follows the sheet dependencies)
1. P1-008 identity model
2. P1-006 API conventions (independent, can run in parallel with 1)
3. P1-010 + P1-011 question domain and answers
4. P1-014 student activity (needs P1-011)
5. P1-015 exposure (needs P1-014)
6. P1-012 explanations
7. P1-013 exam configuration
8. P1-016 parent linking (needs P1-008)

### Product decisions needed before coding
| # | Decision | Default used in this plan |
|---|---|---|
| D1 | Scoring multiple-select answers | All-or-nothing: the selected set must equal the correct set. No partial credit. |
| D2 | Grid-in / student-produced responses (SAT) | Out of scope. Not in P1-010 or P1-011. Track it as a separate point. |
| D3 | Do bookmarked-mode practice sessions respect cooldown? | No. The student chose those questions on purpose. |
| D4 | Does incorrect-retry mode respect cooldown? | Yes, with its own shorter window and a never-starve fallback. |
| D5 | Who must approve when a link is made by invite code? | Nobody further. Creating or redeeming the code is the consent of each side. |
| D6 | Can one account hold both student and parent roles? | No for now. The schema allows it later, but nothing is built for it. |

---

## Point 1: P1-008 Core identity model

**Feature Sheet Requirement:**
Model accounts, profiles, roles, and auth identities: Separate account, profile, role assignment, and external identity records to support student and parent identities. (P0 - Critical | Depends on: P1-003, P1-007)

**Status:** Not started (plan only, no code changes applied)

### Starting point (verified in code)
- `users` holds the account, profile fields (`display_name`, `phone`, `target_test`, `test_experience`), one `role` enum (`student`, `parent`, `admin`), `password_hash` (nullable), `status`, `email_verified_at`.
- Migration `0007` makes email unique per audience through two partial indexes that depend on `users.role`: `users_admin_email_key` (role = admin) and `users_mobile_email_key` (role in student, parent). An email can have one admin account and one mobile account, never both a student and a parent.
- Many places read `users.role`: `AUDIENCE_ROLES` lookups in `users.repository.ts`, `access-token.guard.ts`, `admin-role.guard.ts`, `student-role.guard.ts`, `parent-role.guard.ts`, session records, analytics.
- `student_profiles` already holds academic onboarding data.
- `googleLogin` verifies the Google ID token, then finds the user **by email only** and returns 404 `ACCOUNT_NOT_FOUND` when no account exists. This is deliberate: signup needs role, phone and exam fields. The Google `sub` is never stored.

### Decision
Keep one role per account. Dual student and parent roles (D6) and a `/auth/switch-role` endpoint are **not part of this requirement** and would break the per-audience email rule, the role guards and student-only data. The schema is shaped so that adding more roles later needs no migration.

### Target model
| Table | Purpose | Key / constraints |
|---|---|---|
| `users` | Account anchor: `id`, `email`, `password_hash`, `status`, `email_verified_at`, `last_login_at`, timestamps. `role` stays as the account's **primary role** and is what the audience indexes use. | Existing indexes unchanged |
| `user_profiles` | Person details: `display_name`, `phone`, `avatar_url` (nullable), `target_test`, `test_experience` | `user_id` PK, FK to `users` ON DELETE CASCADE |
| `user_roles` | Role assignments: `user_id`, `role`, `status` (`active`, `revoked`), `granted_at`, `granted_by` (nullable, null = self signup), `revoked_at` | PK `(user_id, role)` |
| `auth_identities` | External identities only: `id`, `user_id`, `provider` (`google`), `provider_user_id`, `email` at link time, `created_at`, `last_sign_in_at` | UNIQUE `(provider, provider_user_id)`, index on `user_id` |

Rules:
1. **Local credentials stay on the account.** Password hash and email OTP remain on `users` / `email_verification_otps`. `auth_identities` is only for federated providers. (Password credentials are reserved separately by P1-009.)
2. **`user_roles` is the assignment record; `users.role` is its primary-role mirror.** Both are written in the same transaction. A **deferred** constraint trigger rejects any `users` row whose `role` has no active matching `user_roles` row. This stops the two from drifting.
   - It must be `CREATE CONSTRAINT TRIGGER ... AFTER INSERT OR UPDATE OF role ON users DEFERRABLE INITIALLY DEFERRED`. `users` is inserted first (to get the id) and `user_roles` second, so an immediate trigger would abort the first statement. Deferred, it is checked at commit.
   - A matching deferred constraint trigger on `user_roles` (`AFTER UPDATE OR DELETE`) re-checks the owning user, so revoking or deleting the primary role row cannot leave `users.role` unmatched.
   - The backfill inserts `users`-derived `user_roles` rows inside one transaction.
3. **One source of truth for profile fields.** `display_name`, `phone`, `target_test`, `test_experience` move to `user_profiles`. The old `users` columns are dropped in the contract migration after the code cutover (see steps). No two-way sync.
4. **Google resolution order:**
   1. Look up `auth_identities` by `(google, sub)`. If found, load that user and sign them in (even if the email changed at Google), **after the audience and status checks in step 4**.
   2. Else look up `users` by the Google email in the `mobile` audience, only if Google reports the email as verified. If found, insert the `auth_identities` row and sign in.
   3. Else return the existing 404 `ACCOUNT_NOT_FOUND` (unchanged behaviour).
   4. **Audience and status checks apply to every path.** Today admins cannot reach `googleLogin` only because `findByEmail(email, 'mobile')` filters to student and parent roles. A lookup by `sub` skips that filter, so after resolving the user by identity the service must check `isRoleInAudience(user.role, 'mobile')` and reject admins (and any non-mobile role) with the same not-found style error, then reject inactive accounts with `ACCOUNT_INACTIVE`. Linking in step 2 stays safe because the lookup is already mobile-only.
   5. Only mobile-audience accounts may have `auth_identities` rows. Google sign-in for the admin panel is a separate decision and is not enabled here.
   6. Keep marking the email verified as today.
7. Linking is race-safe: the unique constraint decides, using `INSERT ... ON CONFLICT DO NOTHING` and a re-read.

### Steps
1. **Migration (expand)**: create `user_profiles`, `user_roles`, `auth_identities`, the sync constraint trigger; backfill profiles and roles from every existing `users` row (all roles, including admin). Down migration drops them.
2. `UsersRepository.create` and the admin signup / `admin-cli.ts` create paths write `users`, `user_profiles` and `user_roles` in one transaction.
3. Move all reads of `display_name`, `phone`, `target_test`, `test_experience` to join `user_profiles` (profile, onboarding, parent dashboard, analytics, mail templates).
4. `auth.service.ts` `googleLogin`: implement the resolution order above.
5. **Migration (contract)**, shipped after step 3 is deployed: drop the moved columns from `users`.
6. Role changes (admin approval, `admin:approve` CLI) update `user_roles` and `users.role` together.
7. Auth responses and JWT keep their current shape. No client change is required. Optionally expose `roles: ['student']` in `GET /auth/me` for future use.
8. Tests: transactional creation (all three rows or none), backfill correctness, sync trigger, Google linking by `sub`, link by verified email, unverified email rejected, 404 for unknown email, concurrent first-time links produce one identity, per-audience email rules still hold, deferred role trigger allows the normal insert order and rejects a commit with no matching role row, an admin account resolved by `sub` is rejected, an inactive account resolved by `sub` is rejected.

### Done when
- Every new account has `users`, `user_profiles` and `user_roles` rows, written transactionally.
- Existing accounts are backfilled and the old profile columns are gone.
- Google login authenticates by `sub` first and falls back to verified-email linking.
- No client app needs to change to keep working.

---

## Point 2: P1-006 API conventions

**Feature Sheet Requirement:**
Implement API response, pagination, and error conventions: Standardize request IDs, validation errors, cursor pagination, idempotency where needed, and consistent timestamps. (P1 - High | Depends on: P1-005)

**Status:** Not started (plan only, no code changes applied)

### Starting point (verified in code)
- Request IDs, the validation error shape, the response envelope and ISO UTC timestamps are already standard (`common/`, `GlobalExceptionFilter`).
- All paginated endpoints use `page`, `pageSize`, `total` (`common/schemas/pagination.schemas.ts`). Admin tables and the mobile review tabs rely on `total` for badges.
- Many mutations are **already idempotent**: practice answers are write-once (`selected_option_id IS NULL` guard), starting a mock exam resumes the open attempt (`one_open_mock_attempt_idx`), submit is a no-op when repeated, bookmarks use PUT/DELETE.
- `architecture.md` section 20 states that no interceptors are used.

### Approach

**A. Pagination: dual mode, no breaking change**
- Keep offset pagination (`page`, `pageSize`, `total`) wherever clients need totals: admin tables, review-tab badges.
- Add keyset cursor pagination for list endpoints that behave like feeds and can grow without bound.
- Dual-mode endpoints accept either `cursor` + `limit` or `page` + `pageSize`. If `cursor` is present, use keyset; otherwise offset. Offset responses stay exactly as today.
- New helper `common/pagination/cursor.ts`: opaque base64url cursor containing the sort key plus `id` as a tiebreaker; signed with an HMAC derived from an existing secret so tampering is rejected with a 400 validation error.
- Cursor response shape: `{ items, nextCursor: string | null, hasMore: boolean }`. `total` is returned only in offset mode.
- First adopters: `GET /student/mock-exams/history`, `GET /student/stats/questions`, and the admin analytics recent-activity lists. Future feed endpoints (audit log, P1-059) use the helper from day one.
- Pagination bounds stay `MAX_PAGE_SIZE = 100`.

**B. Idempotency: backend-first, optional header, small scope**
- Table `idempotency_keys`: `id`, `user_id`, `key`, `method`, `route` (route template, not raw path), `request_hash` (SHA-256 of the canonical body), `state` (`in_progress`, `completed`), `status_code`, `response_body` (jsonb), `created_at`, `expires_at`. UNIQUE `(user_id, key, method, route)`.
- `@Idempotent()` decorator with a Nest interceptor. Record the exception to the "no interceptors" rule in `architecture.md` section 20.
- Header `Idempotency-Key` is optional. Without it the request runs as today.
- With a header:
  1. Atomically insert an `in_progress` row (`ON CONFLICT` read the existing row).
  2. Existing `completed` row with the same hash: replay the stored status and body, add `Idempotent-Replayed: true`.
  3. Same key, different hash: 422 `IDEMPOTENCY_KEY_REUSED`.
  4. Same key still `in_progress`: 409 `IDEMPOTENCY_IN_PROGRESS`. A stuck `in_progress` row older than 60 seconds may be taken over.
  5. Store only 2xx responses. On any other outcome delete the row so the client may retry.
- Never apply it to endpoints that return tokens or one-time secrets (auth endpoints, invite creation that returns a code).
- Apply it only where the operation is not naturally idempotent: bulk question import, admin create endpoints (question, exam, taxonomy node, passage), study-plan generate and item create, parent request creation.
- Purge rows past `expires_at` (24 hours) using the same scheduler pattern as `AnalyticsSchedulerService`.
- Document the header and the two error codes in Swagger.

**C. Timestamp and error consistency**
- Add a unit test that fails if a response schema declares a timestamp that is not ISO 8601 UTC.
- Add the new error codes to `shared/constants/http.constants.ts`.

### Clients
- No client must change. Optionally send a UUID `Idempotency-Key` from the admin bulk-import and create forms (the only places double submit is realistic). Mobile and student web need nothing because their endpoints are already safe.

### Steps
1. `common/pagination/cursor.ts` plus unit tests (ordering, ties, tampering, expired format).
2. Cursor schemas and a dual-mode query schema helper next to `paginationFields`.
3. Adopt in the three first endpoints.
4. Migration for `idempotency_keys`, decorator, interceptor, purge job.
5. Apply `@Idempotent()` to the endpoints listed above.
6. Tests: keyset stable under inserts, tampered cursor 400, offset responses unchanged, same-key replay, body mismatch 422, concurrent same key 409, failed request not cached, header omitted pass-through, replay does not create a duplicate row.

### Done when
- Feed endpoints support stable cursor pagination while admin tables and review badges keep working unchanged.
- Duplicate or replayed non-idempotent requests with a key create no duplicate records.
- Existing clients behave exactly as before.

---

## Point 3: P1-010 Question domain model and P1-011 Question answers

**Feature Sheet Requirements:**
- **P1-010**: Model subjects, topics, categories, question types, and severity: Create normalized taxonomy supporting subject/topic/category hierarchy, question type, and Easy/Medium/Difficult severity. (P0 - Critical | Depends on: P1-007)
- **P1-011**: Model flexible single and multiple correct answers: Use a question-option association and correct-option mapping that supports one or many correct options without schema change. (P0 - Critical | Depends on: P1-010)

**Status:** Not started (plan only, no code changes applied)

### Starting point (verified in code)
- Taxonomy exists: `taxonomy_nodes` (`section` → `domain` → `skill`, exam-specific) and `question_revision_skills`. Difficulty exists as 1/2/3 (Easy/Medium/Difficult).
- The **option association already supports many correct options**: `question_options.is_correct` is per option. The blocker is only the publish trigger `validate_question_revision_for_publish()` and the app check in `questions.service.ts` (`correctCount !== 1`), plus the bulk schema (`correct: z.enum(['A'..'E'])`).
- A student's answer is one column, `assessment_attempt_items.selected_option_id`. Correctness is computed in SQL from that single option (`i.is_correct = o.is_correct`).
- "Answered" is detected by `selected_option_id IS NOT NULL` in many queries (practice `SEEN`, `LATEST_ANSWER`, stats, mock exam progress, home, parent dashboard, analytics, study plan).
- There is no question-type field.

### Decision
Scope is **single-choice and multiple-select only** (D2: grid-in is a separate feature). Whether a question is single or multiple is a stored `question_type`, validated against the number of correct options, so the type is explicit for filtering and clients, and the correct-option mapping needs no schema change.

### Question model (P1-010)
- Enum `question_type`: `single_choice`, `multiple_select` (extensible later).
- `question_revisions.question_type question_type NOT NULL DEFAULT 'single_choice'`. Backfill is a no-op because every existing question is single choice.
- Publish rules, enforced in the rewritten trigger and in the service:
  - `single_choice`: 2 to 5 options, exactly 1 correct.
  - `multiple_select`: 3 to 5 options, at least 2 correct, and fewer correct than total options.
- The existing taxonomy (subject → topic → category → skill) and the 3 severity levels already satisfy the hierarchy and severity parts. No taxonomy change. The plan only confirms them in `architecture.md` and exposes `questionType` in admin and student responses.
- Shuffling (`shuffle_options`, `option_order`) works unchanged for both types.

### Answer storage (P1-011)
- New table `assessment_attempt_item_options (item_id uuid REFERENCES assessment_attempt_items ON DELETE CASCADE, option_id uuid REFERENCES question_options ON DELETE RESTRICT, PRIMARY KEY (item_id, option_id))`. This is the **single source of truth** for what a student selected, for both types.
- `assessment_attempt_items.is_correct` stays as the stored result of grading (set-equality: selected set equals correct set; D1).
- **Expand/contract for `selected_option_id`:**
  1. Expand: add the new table, backfill one row per existing answered item, and for a transition period keep writing `selected_option_id` (the first selected option by position) so old queries and old app builds keep working.
  2. Switch every read of "answered" and "selected" to shared SQL helpers built on the new table.
  3. Contract: drop `selected_option_id` in a later migration once no code reads it.
- Audit by grepping every use of `selected_option_id` and `is_correct` before the switch. Known users: `practice.repository.ts` (`LATEST_ANSWER`, `SEEN`, answer save, `correctOptionId` subquery), `stats.repository.ts`, `mock-exams.repository.ts` (answer save, `scoreBySubject`, progress counts, review), plus home, parent dashboard, analytics academic and study-plan queries.
- The practice rule "an answer is saved once" is kept, with an **atomic claim of the item row**, not a separate existence check:
  1. In one transaction, run `UPDATE assessment_attempt_items SET answer_changed_at = now(), selected_option_id = <first selected option> WHERE id = $item AND answer_changed_at IS NULL AND <session in progress, owned by the student, not expired>`. The row lock makes concurrent taps wait, so only the first succeeds; zero rows updated means "already answered" (or not allowed).
  2. Only after the claim succeeds, insert the selected set into `assessment_attempt_item_options` and set `is_correct` by comparing that set with the correct set.
  3. A `NOT EXISTS` check on the selections table alone is not a safe guard: two simultaneous taps with different option sets would both pass it and both insert, because the primary key `(item_id, option_id)` only stops identical options.
  4. During the expand phase, writing `selected_option_id` in the claim update means the old `selected_option_id IS NULL` logic and old queries keep working until the column is dropped.
- The mock exam rule "an answer can change until the section locks" is kept: replace the selected set in one transaction (delete then insert), update `is_correct` and `answer_changed_at`.

### API
- Answer endpoints (`PUT .../items/:itemId/answer`, practice and mock) accept `{ optionIds: string[] }`. The old `{ optionId }` is still accepted and treated as `optionIds: [optionId]` so old builds keep working. Validation: ids belong to the item, no duplicates, exactly 1 for `single_choice`, 1 to option count for `multiple_select`.
- Question delivery adds `questionType` and, for `multiple_select`, a flag the client shows as "Select all that apply". It never reveals how many options are correct.
- Feedback and review add `correctOptionIds[]` and `selectedOptionIds[]`. The old singular `correctOptionId` and `selectedOptionId` stay populated for `single_choice` during the transition.
- Admin question create/update schemas add `questionType`; each option keeps `isCorrect`. Bulk import `correct` accepts one key or a comma list (`A,C`), and the import template documents it.

### Admin panel (`learning-app-admin`)
- Question form: type selector; radio for single choice, checkboxes for multiple select; inline validation matching the backend rules.
- Question list and exam question picker: type column and type filter; preview matches the type.
- Bulk import parser, template and error messages accept `A,C`.

### Mobile (`lm`) and Student Web (`learning-web-app`)
- One reusable question component handles both types: radio behaviour for single choice, checkbox behaviour for multiple select.
- Practice: single choice keeps tap-to-submit. Multiple select shows an explicit "Check answer" button.
- Mock exams: selecting and clearing autosaves the set as it does today.
- Results and review render several selected and correct options.
- Old builds only ever receive `single_choice` questions unless an admin creates multiple-select ones, so release the clients first, then enable multiple-select in admin.

### Steps
1. Migration: enum, `question_type` column, new trigger function, selections table, backfill.
2. Update question DTOs, Zod schemas, `questions.service.ts` checks, bulk service and template.
3. Shared SQL helpers for "answered" and "selected set"; migrate all queries.
4. Update grading, answer endpoints, response schemas and Swagger.
5. Admin, mobile and web changes.
6. Contract migration dropping `selected_option_id` after rollout.
7. Tests: trigger rules for both types, bulk `A,C`, set-equality grading (exact match, subset, superset, empty), write-once practice (including two concurrent taps with different option sets: only the first is saved), mock re-answer replaces the set, old `{ optionId }` payload, shuffle mapping with several options, stats and review show no duplicate or missing rows.

### Done when
- Admins can create, bulk import and publish single-choice and multiple-select questions.
- Students on mobile and web can answer, get feedback and review both types.
- Scoring and stats are correct with no duplicated or dropped rows, and old clients keep working.

---

## Point 4: P1-012 Explanations

**Feature Sheet Requirement:**
Add explanation schema and content lifecycle: Store explanation body, media references, draft/published state, and missing-content flag so content can be added later. (P1 - High | Depends on: P1-010)

**Status:** Not started (plan only, no code changes applied)

### Starting point (verified in code)
- `question_revisions.explanation_markdown` exists, and explanation images use `question_revision_assets` with `usage = 'explanation'` (the column is `usage`, not `asset_usage`).
- There is **no separate publish endpoint**. Publishing happens through the `publish` flag on create, update and bulk import. Editing a published question already creates a **new revision that is published immediately** while the old one is retired. So "add the explanation later" already works through the normal edit path.
- Nothing reports a missing explanation.

### Decision
The lifecycle follows the revision: an explanation is `draft` while its revision is a draft and `published` once the revision is published. That satisfies the draft/published requirement without a second state machine. The missing flag is **computed**, not stored, so it can never be stale.

### Approach
1. **Derived fields** (SQL expression shared by all queries):
   - `hasExplanation` = `btrim(coalesce(explanation_markdown, '')) <> ''` **or** an explanation asset exists (`question_revision_assets.usage = 'explanation'`).
   - `explanationStatus` = `missing` when `hasExplanation` is false, otherwise `draft` or `published` from the revision status.
2. **Publishing is never blocked.** Create, update and bulk responses add `warnings: ['MISSING_EXPLANATION']` when the published revision has no explanation. Bulk results carry the warning per row plus a summary count.
3. **Admin API**: `GET /admin/questions` gains `hasExplanation=true|false` and returns `hasExplanation` / `explanationStatus` per row. Add `warnings` to the response Zod schemas on both the backend (`question-response.schemas.ts`) and the admin (`content.schemas.ts`).
4. **Student API**: responses return `explanation` (null when missing) and `explanationAvailable: boolean`. An explanation is shown only for questions the student has answered (practice) or after the attempt is finalized (mock review), as today.
5. **Edit path**: when only the explanation text or image changes on a published question, keep the current "new revision" behaviour (it keeps past attempts pinned to the content the student saw). No new endpoint.

### Admin panel
- "Missing explanation" filter and a badge on the questions list.
- A non-blocking warning toast after saving a published question without an explanation.
- The import preview and result show the missing-explanation count.

### Mobile and Student Web
- Show "Explanation coming soon" when `explanationAvailable` is false, in practice feedback and in exam review.

### Steps
1. Shared SQL expression and repository changes; add `warnings`, `hasExplanation`, `explanationStatus` to schemas.
2. Questions service returns warnings from create, update and bulk.
3. Add the `hasExplanation` list filter.
4. Admin UI and client empty states.
5. Tests: whitespace-only body counts as missing, an image alone counts as present, publish succeeds with the warning, filter returns the right rows, bulk summary count, student payload never leaks an explanation before answering.

### Done when
- Admins can find every question missing an explanation and are warned on publish.
- A question can be published without one and gets one later through a normal edit.
- Mobile and web show a clean placeholder.

---

## Point 5: P1-013 Exam configuration

**Feature Sheet Requirement:**
Model exam templates and category weightage: Model configurable exam question count, duration, eligibility filters, and category-level weightage; isolate it from practice selection. (P0 - Critical | Depends on: P1-010)

**Status:** Not started (plan only, no code changes applied)

### Starting point (verified in code)
- `assessment_templates` and `assessment_template_versions` carry timing mode (`overall` or `per_section`), duration and `shuffle_questions`. Sections are `assessment_modules` with `duration_seconds`.
- Sections get their questions from hand-picked rows in `assessment_module_questions`. `exams.service.ts` `resolveSections()` requires a non-empty `questionIds` for every section.
- Template versions follow a draft → published lifecycle and are immutable once published.

### Decision
P1-013 is the **model, admin configuration and validation**. Choosing questions for an attempt is the composition engine of P1-044, which depends on this point and on cooldown (P1-015). So blueprint sections store **rules**, and questions are resolved **per attempt at start time**, not frozen at draft time. Freezing one rolled set would give every student identical questions and would make per-student cooldown impossible.

### Schema
- `assessment_modules.assembly_mode text NOT NULL DEFAULT 'manual'` with CHECK (`manual`, `blueprint`). `manual` is today's behaviour and stays fully supported.
- `assessment_modules.target_question_count smallint` (required when `blueprint`; for `manual` it equals the number of picked questions).
- `assessment_module_blueprint_rules`:
  - `id`, `module_id` → `assessment_modules` ON DELETE CASCADE
  - `taxonomy_node_id` → `taxonomy_nodes` (the category or skill this weight applies to)
  - `weight_percent numeric(5,2)` with CHECK `> 0`
  - `difficulties smallint[]` (eligible levels, default all)
  - `question_types question_type[]` (eligible types, default all)
  - UNIQUE `(module_id, taxonomy_node_id)`
- Per-rule question counts are **derived** from the module target and the weights (largest-remainder rounding, always summing to the target). They are not stored, so weights and counts can never disagree.
- Rules belong to the template version, so a published version's blueprint is immutable.
- `exclude recently used` filters are **not** modelled here: recency is the cooldown from P1-015, applied at composition.

### Isolation from practice
- Practice selection never reads blueprint tables, and blueprint resolution never reads practice settings. A test asserts that practice queries do not touch the new tables.

### API and validation
- Admin exam create/update accepts per section: `assemblyMode`, `targetQuestionCount`, `blueprint: [{ nodeId, weightPercent, difficulties, questionTypes }]`; `questionIds` is required only for `manual`.
- Validation: weights sum to 100 within 0.01; every node belongs to the exam's taxonomy; target count is at least the number of rules; per-section duration rules unchanged.
- **Availability preview**: `POST /admin/mock-exams/:examId/blueprint-preview` returns, per rule, the derived question count and how many published, unarchived questions are eligible, with a `satisfiable` flag. It is read-only and creates nothing.
- **Publish validation**: a blueprint section can be published only if every rule is satisfiable right now. Question availability can shrink later, so the composer (P1-044) must handle shortfalls at attempt time (fail clearly or fall back, decided there).
- **Publish gate**: until the P1-044 composer exists, publishing an exam with a blueprint section returns 409 `BLUEPRINT_COMPOSER_NOT_AVAILABLE`. Admins can build and preview blueprints but cannot publish an exam that students could not actually take.

### Admin panel
- Section editor: mode toggle (Manual, Blueprint), target count, weights table by category with a live "must equal 100" total, difficulty and type filters per rule, and a preview panel showing eligible counts.
- Exam detail shows the blueprint summary.

### Mobile and Student Web
- No change. Students keep reading `assessment_module_questions` until P1-044 makes the composer populate it at attempt start.

### Steps
1. Migration for the new columns and table, plus down migration.
2. Update exam DTOs, schemas, `resolveSections()` and the repository.
3. Preview endpoint and publish validation with the temporary gate.
4. Admin UI.
5. Tests: weight sum validation, count derivation (rounding sums to target), filters, taxonomy ownership check, manual sections unchanged, published versions immutable, gate returns 409, practice isolation.

### Done when
- Admins can configure question count, duration, eligibility filters and category weights, and see availability before publishing.
- Manual exams behave exactly as before.
- Blueprint rules are stored per version and ready for the P1-044 composer.

---

## Point 6: P1-014 Student activity

**Feature Sheet Requirement:**
Model attempts, answers, bookmarks, tests, and progress: Create append-safe records for question attempts, answer selections, bookmarks, test sessions, test answers, and score snapshots. (P0 - Critical | Depends on: P1-011)

**Status:** Not started (plan only, no code changes applied)

### Starting point (verified in code)
- `assessment_attempts` stores `raw_score` / `scaled_score` and is updated in place.
- Practice answers are already write-once. **Mock exam answers change in place**, so earlier answers and timing are lost.
- Item rows pin an immutable `question_revision_id`, so question content cannot drift under past attempts.
- Scores shown after the fact are computed live from items (`scoreBySubject`).
- Bookmarks (`question_bookmarks`) are deleted on removal, leaving no history.

### Approach

**A. Answer events (append-only)**
- `assessment_attempt_answer_events`: `id`, `attempt_item_id` → items ON DELETE CASCADE, `event_type` (`answered`, `changed`, `cleared`, `flagged`, `unflagged`), `is_flagged boolean`, `time_spent_seconds`, `created_at`.
- Selected options of an event go in `assessment_attempt_event_options (event_id, option_id)`. This keeps a real foreign key to `question_options` and is consistent with the P1-011 selections table. (No uuid array column.)
- An event is written **only when the answer set or the flag actually changes** compared with the current item state, inside the same transaction as the item update. Autosaves that repeat the same data write nothing.
- Events are never updated or deleted by application code. A shared trigger function `prevent_direct_event_mutation()` is attached to every append-only table in this plan (answer events, event options, score snapshots, bookmark events, and the P1-016 link events):
  - `UPDATE` always raises.
  - `DELETE` raises unless it comes from a foreign-key cascade, detected with `pg_trigger_depth() > 1`. PostgreSQL fires row-level `BEFORE DELETE` triggers during cascades, so a trigger that always raised would make every delete of a user or attempt (and test cleanup that deletes) fail.
  - The depth check is a loose guard (another trigger that deletes would also pass), which is acceptable here. Row-level triggers do not fire on `TRUNCATE`, so test cleanup that truncates is unaffected; add a statement-level `TRUNCATE` trigger only if truncation must be blocked too.
  - Tests cover: direct update rejected, direct delete rejected, deleting a user or attempt cascades cleanly.
- The item row stays as the **current state** for fast reads. Events are the history.

**B. Score snapshots (immutable)**
- `assessment_attempt_score_snapshots`:
  - `attempt_id` PK → `assessment_attempts` ON DELETE CASCADE
  - `raw_score`, `scaled_score` (nullable), `correct_count`, `incorrect_count`, `unanswered_count`
  - `breakdown jsonb` (per section and per subject or category: correct, total), validated by a Zod schema in code
  - `completion_reason` (`submitted`, `timed_out`)
  - `scoring_version text` (starts at `v1`)
  - `finalized_at`, `created_at`
- Written in the same transaction as `finishAttempt()` for mock exams (both `submitted` and `expired`) and in `submitAttempt()` for practice sessions. The shared `prevent_direct_event_mutation()` trigger blocks UPDATE and direct DELETE (cascade deletes from the attempt are allowed).
- Backfill a snapshot for every existing submitted or expired attempt from its items, with `scoring_version = 'backfill-v0'`.
- Result and history endpoints read the snapshot. Question-level review keeps joining the pinned revision, which is immutable.
- `abandoned` is not a completion reason today because no abandon flow exists. Add it when such a flow is built.

**C. Bookmarks**
- Keep `question_bookmarks` as the current state (what the app lists).
- Add `question_bookmark_events (id, user_id, question_id, action 'added'|'removed', created_at)`, written in the same transaction as each add or remove (only when state really changes). This gives an append-safe history without changing how bookmarks work.

**D. Test sessions, test answers, progress**
- Test sessions (`assessment_attempts` with modules) and test answers (items plus selections) already exist and are covered by A and B. Progress aggregates keep reading current state and snapshots.

### Clients
- No payload change is required. Result and history screens already show scores; they simply read from stable data.

### Steps
1. Migration: event tables, snapshot table, bookmark events, immutability triggers, snapshot backfill.
2. Repositories: write events on practice and mock answer saves and on flag changes; write the snapshot in the finish and submit paths.
3. Switch result and history queries to snapshots.
4. Tests: no event on a repeated identical autosave, one event per real change, events survive a later change, snapshot written for submit and for timeout, snapshot unchanged after the question is edited or retired, UPDATE and DELETE on events and snapshots rejected, bookmark add/remove history, backfill matches the old live scores.

### Done when
- Every real answer or flag change is kept in an append-only history for single and multiple-select answers.
- Every finished attempt has an immutable score snapshot and results never change afterwards.
- Bookmark changes are recorded without altering current bookmark behaviour.

---

## Point 7: P1-015 Exposure management

**Feature Sheet Requirement:**
Model question exposure and cooldown: Track question delivery and attempt timestamps per student; support cooldown eligibility queries and configurable windows. (P0 - Critical | Depends on: P1-014)

**Status:** Not started (plan only, no code changes applied)

### Starting point (verified in code)
- Practice selection orders by `SEEN` (answered before in a practice session, mock exams not counted), then `random()`. There is no delivery tracking, no timestamps used, no cooldown, no configuration.
- Practice items are created at session start; mock exam questions are delivered module by module.
- The `attempt_items` unique key (`module`, `question_revision`) means autosaves cannot create new items.

### Approach

**A. Exposure table `student_question_exposures`**
- `user_id` → `users` ON DELETE CASCADE, `question_id` → `questions` ON DELETE CASCADE, PRIMARY KEY `(user_id, question_id)`.
- Delivery: `first_delivered_at`, `last_delivered_at`, `delivered_count`.
- Attempt outcome: `last_answered_at` (nullable), `answered_count`, `last_was_correct` (nullable).
- `last_source` (`practice`, `mock_exam`), `last_attempt_id` (guards against double counting when a step is retried).
- Index on `(question_id)` for admin metrics and on `(user_id, last_delivered_at)`.

**B. When it is written**
- **Delivery** is recorded when an item is created or becomes visible: practice session creation (all its items), mock exam module start (that module's items). Abandoned attempts therefore still count as delivered. The upsert increments `delivered_count` only when `last_attempt_id IS DISTINCT FROM` the current attempt.
- **Outcome** is recorded when an answer is saved in practice, and for every item of a mock exam when it is finalized (submitted or expired). Unanswered items leave `last_was_correct` null.
- Autosaves never change counts.
- Backfill from existing items and attempts.

**C. Configurable windows**
- Table `question_cooldown_settings` per `exam_version_id`: `after_correct_days`, `after_incorrect_days`, `after_unanswered_days`, `retry_after_incorrect_days` (for the incorrect-retry queue), `updated_by`, timestamps. Seeded defaults: 14 / 3 / 7 / 1. Admin can edit them through an admin endpoint and screen. Missing row means seeded defaults.
- A question is **eligible** for a student when it was never delivered, or `now()` is past the last outcome time plus the window that matches the last outcome.

**D. Selection rules (practice)**
- Custom mode, order of preference: unseen → past cooldown (oldest first) → still in cooldown (oldest first). Random tie-breaker inside each group. The pool never starves: the last group is only used when earlier groups cannot fill the requested count.
- Bookmarked mode ignores cooldown (D3).
- Incorrect-retry mode uses `retry_after_incorrect_days` with the same never-starve fallback (D4).
- The practice response can include `reusedCount` so the app can tell the student some questions were repeated early.
- The eligibility logic is one reusable SQL fragment, also used by the P1-044 exam composer.

**E. Admin metrics**
- `GET /admin/questions` gains `deliveredCount` and `studentCount` per question (aggregate from the exposure table, paginated and indexed) and an `overExposed` filter with a threshold parameter.
- Exposure data is **never** returned to students or parents.

### Clients
- Mobile and web: no required change. Optionally show the "some questions repeated" notice.
- Admin: cooldown settings screen, exposure columns and filter.

### Steps
1. Migration: exposure table, settings table with seeds, backfill.
2. Write paths in practice session creation, practice answer save, mock module start and mock finalization.
3. Shared eligibility fragment and the new selection queries for practice custom and incorrect modes.
4. Admin endpoints and UI.
5. Tests: retried request does not double count, autosave does not inflate, abandoned mock counts as delivered, cooldown windows per outcome, never-starve with a tiny bank, bookmarked ignores cooldown, incorrect mode window, settings change takes effect, students cannot see exposure fields.

### Done when
- Delivery and outcome timestamps are recorded for practice and mock exams per student.
- Practice selection applies configurable cooldown without ever failing for lack of questions.
- Exposure counts stay accurate under retries and autosaves.

---

## Point 8: P1-016 Parent linking

**Feature Sheet Requirement:**
Model parent-child invitations and links: Add invite token, status, expiry, acceptance, revocation, and parent-child relationship records. (P0 - Critical | Depends on: P1-008)

**Status:** Not started (plan only, no code changes applied)

### Starting point (verified in code)
- `parent_child_links` has `UNIQUE (parent_id, child_id)` and statuses `pending`, `approved`, `rejected`. A parent request is made by the child's email; the student approves or rejects.
- A rejected row **stays** and holds the 24 hour re-request cooldown. `reopenRejected` updates that same row back to `pending`.
- A parent ending a link hard-deletes the row (`deleteForParent`). A student ends an approved link by "rejecting" it.
- The child limit counts links where `status <> 'rejected'`.
- The `link` throttler policy already rate-limits parent lookup and request.
- Invites with no known recipient cannot live in `parent_child_links` (both ids are required).

### Decision
- **Keep the pair unique constraint.** The existing logic reuses one row per pair, and that stays. History is kept in an append-only events table instead of extra rows, which avoids rewriting the cooldown and reopen logic.
- Add a `revoked` status instead of deleting rows.
- Invitations get their own table.

### Schema
1. `parent_child_links`: allow status `revoked`; add `revoked_at timestamptz`, `revoked_by uuid REFERENCES users`; update the status and `responded_at` CHECKs accordingly. The pair constraint stays.
2. `parent_student_invites`:
   - `id`, `creator_id` → `users`, `creator_role` (`student` or `parent`)
   - `token_hash` (SHA-256 of a 128-bit random token), UNIQUE
   - `expires_at` (default now + 7 days)
   - `status` (`active`, `redeemed`, `cancelled`). **Expired is derived** from `expires_at`; no job is needed to flip it
   - `redeemed_by` → `users`, `redeemed_at`, `link_id` → `parent_child_links`
   - Index on `(creator_id, status)`
3. `parent_link_events` (append-only): `id`, `link_id`, `event` (`requested`, `approved`, `rejected`, `revoked`, `invite_created`, `invite_redeemed`, `invite_cancelled`), `actor_id`, `created_at`. UPDATE and direct DELETE rejected by the shared `prevent_direct_event_mutation()` trigger (defined in P1-014; cascade deletes from the link or user are allowed). This is also the first building block for the audit trail in P1-059.

### Behaviour
- **Invite creation**: the creator gets the raw token once (shown as a code and as a link); only the hash is stored. Creating a new invite cancels the creator's previous active invite (this is "resend").
- **Redeem** (D5): the redeemer must hold the opposite role from the creator and cannot be the creator. A valid, unexpired, active invite creates the link as `approved` straight away (or turns an existing `pending`, `rejected` or `revoked` row for that pair into `approved`). The invite becomes `redeemed`. Child limit is enforced. All of it runs in one transaction; the unique constraint decides races.
- **Revoke**: parent `DELETE /parent/children/:linkId` becomes a soft revoke (status `revoked`, event written). The student revokes with `POST /student/parent-requests/:linkId/revoke`. Revoking ends data access immediately because every parent query already requires `status = 'approved'`. After a revoke the same pair can link again through a request or an invite.
- **Existing email-request flow stays unchanged.** Student rejection and the 24 hour cooldown keep working. Student "reject" on an approved link is replaced by the explicit revoke, which records `revoked` instead of `rejected`.
- Child limit now counts only `pending` and `approved`.
- Redeem attempts are rate-limited with the `link` policy, plus a short per-account failure lockout, because the token is the only secret.

### Endpoints (role-scoped, versioned like the rest of the API)
| Endpoint | Who | Purpose |
|---|---|---|
| `POST /student/parent-invites` | student | Create an invite code for a parent |
| `GET /student/parent-invites` | student | List own invites with derived status |
| `DELETE /student/parent-invites/:id` | student | Cancel an active invite |
| `POST /student/parent-invites/redeem` | student | Redeem a code created by a parent |
| `POST /parent/invites` | parent | Create an invite code for a child |
| `GET /parent/invites` | parent | List own invites |
| `DELETE /parent/invites/:id` | parent | Cancel an active invite |
| `POST /parent/invites/redeem` | parent | Redeem a code created by a student |
| `POST /student/parent-requests/:linkId/revoke` | student | End an approved link |
| `DELETE /parent/children/:linkId` | parent | End a link (now a soft revoke) |

Invite creation is not made idempotent through the P1-006 mechanism because it returns a secret.

### Clients (`lm` mobile and `learning-web-app` web)
- Student: "Invite a parent" generates and shows the code or share link; list of invites with status; list of linked parents with a Revoke button; field to enter a parent's code.
- Parent: "Enter invite code" next to the existing request-by-email; invite creation for a child; status indicators (`pending`, `approved`, `expired`, `revoked`).

### Steps
1. Migration: new status and columns, invites table, events table with immutability trigger.
2. Repository and service changes: revoke, invite create, list, cancel, redeem; event writes in every transition; child-limit change.
3. Controllers, Zod schemas, Swagger.
4. Mobile and web UI.
5. Tests: token stored hashed only, expiry derived and enforced, wrong-role and self redeem rejected, double redeem rejected, concurrent redeem yields one link, redeem reuses a revoked or rejected row, revoke cuts dashboard access immediately, relink after revoke, child limit ignores revoked, events recorded and immutable, rate limit applied.

### Done when
- Parents and students can link with an invite code from either side, and the email request flow still works.
- Unclaimed invites expire without touching link tables, and old invites purge safely.
- Revocation keeps the history, ends access at once and allows a clean re-link of the same pair.
- Mobile and web support invite, redeem and revoke.

---

## What changed from the previous version of this plan
| Point | Change |
|---|---|
| P1-008 | Dropped dual roles and switch-role; kept `users.role` as the primary-role mirror guarded by a trigger; single source of truth for profile fields; Google logic kept with the existing 404 rule. |
| P1-006 | Idempotency scoped to non-idempotent endpoints, backend-first, no mandatory client work; signed cursors; interceptor exception documented. |
| P1-010/011 | Removed grid-in; one selections table as the single answer store; expand/contract for `selected_option_id`; full query audit; backward-compatible `optionId`. |
| P1-012 | Fixed the column name; lifecycle follows the revision; computed missing flag; uses the existing edit-creates-revision path. |
| P1-013 | Rules resolved per attempt, not frozen in draft; derived counts; no recency filter; publish gate until the P1-044 composer exists. |
| P1-014 | Event options in a table instead of an array; immutability triggers; backfilled snapshots; no unused `abandoned` reason; bookmark events. |
| P1-015 | Tracks delivery, not only completion; configurable windows in a table; cooldown rules per mode; never-starve ordering. |
| P1-016 | Kept the pair constraint and added `revoked` plus an events table; derived expiry; role-scoped endpoints; child limit fix; explicit redeem semantics. |
