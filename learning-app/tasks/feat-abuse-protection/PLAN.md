# End-to-End Dependency and Execution Plan: P1-058 Abuse Protection

## 1. Full Dependency Chain Overview

Case P1-058 (Abuse Protection) is the target case of this plan. In the Phase 1 master workplan (`phase1_student_exam_prep_workplan.xlsx`), it sits in the Security & Compliance workstream and depends directly on three foundational feature implementations:
- **P1-058** depends directly on:
  1. **P1-018** (Email OTP Authentication) in Authentication & Accounts.
  2. **P1-054** (Parent-Child Invite Workflow) in Parent Experience.
  3. **P1-027** (Question Bulk Import Parser and Staged Validation) in Question Bank & Content.
- In addition, the official detailed description of P1-058 explicitly mandates rate-limiting for **answer submission endpoints** and **sign-in endpoints**, linking it to student exam/practice test workflows:
  4. **P1-032** (Practice Session Answer Submission and Evaluation) in Practice Tests.
  5. **P1-037** (Mock Exam Attempt Answer Submission and Timing) in Mock Exams.
  6. **P1-017** / **P1-019** (Google Sign-In, Session Lifecycle, and Revocation) in Authentication & Accounts.

All these feature modules rest upon the Phase 1 data modeling and service foundation:
- **P1-018** depends on **P1-008** (Core Identity Model) and **P1-005** (Service Bootstrap).
- **P1-054** depends on **P1-016** (Parent Linking Model), **P1-019** (Session Revocation), and **P1-020** (Authorization).
- **P1-027** depends on **P1-026** (Canonical Import Template), which in turn depends on **P1-010** (Question Model), **P1-011** (Answers), and **P1-012** (Explanations).
- All database entities depend on **P1-007** (Database Schema and Migration Baseline).
- Service architecture and permissions depend on **P1-001** (Phase 1 Scope), **P1-002** (Technical Architecture), and **P1-003** (Role and Permission Matrix).

```
                            P1-001: Phase 1 Scope
                          (Discovery & Architecture)
                                 /     |     \
                                /      |      \
                               v       v       v
      P1-002: Architecture   P1-003: Roles   P1-004: NFR Baseline
               |                   |                  |
               v                   |                  |
      P1-007: DB Schema            |                  |
        /      |      \            |                  |
       /       |       \           |                  |
      v        |        v          v                  |
   P1-010-012  |    P1-008: Core Identity             |
 (Question &   |    (Accounts, Profiles, Roles)       |
   Answers)    |        /             \               |
      |        |       /               \              |
      v        v      v                 v             |
   P1-026    P1-016: Parent        P1-005: Service Bootstrap
      |      (Invite & Link Model)   (Modular Backend Baseline)
      v               |               /       \
   P1-027: Import     |              /         \
   (Parser/Staging)   |             v           v
          |           |      P1-018: OTP     P1-017 / P1-019
          |           |      (Email Auth)    (Sessions & Tokens)
          |           |             \           /
          |           \              \         /
          |            \---> P1-054: Parent Invites
          |                        |
          +------------------------+-----------------------+
                                   |                       |
                                   v                       v
                        P1-058: Abuse Protection    P1-032 / P1-037
                       (Rate Limits, Anti-Automation, (Answer Submissions)
                         Suspicious Behavior Logs)         |
                                   ^                       |
                                   +-----------------------+
```

In this plan, **100% completion of any case means bringing its code and documentation to the "Ready for Review" status**, with all acceptance criteria satisfied, edge cases protected, and automated test suites passing.

---

## 2. Code-Based Status Evaluation of the Entire Chain

The statuses below are evaluated strictly against what currently exists in the live repositories (`learning-app-backend`, `learning-app-admin`, `learning-web-app`, and `D:\lm`):

| Case ID | Workstream | Priority | Code-Based Status | What Exists in Code Today | What Is Missing to Reach 100% (Ready for Review) |
|---|---|---|---|---|---|
| **P1-001** | Discovery & Architecture | P0 - Critical | **Ready for Review** (100%) | Phase 1 scope, boundaries (iOS priority, web/admin readiness), and core features documented in `architecture.md` (§1, §24, §25). | None. Scope boundaries are established and active. |
| **P1-002** | Discovery & Architecture | P0 - Critical | **Ready for Review** (100%) | Layered monolith, service boundaries, and URI versioning (`/api/v1/`) active in `architecture.md` and `src/shared/http/configure-http.ts`. | None. Architectural baseline is active. |
| **P1-003** | Discovery & Architecture | P0 - Critical | **Ready for Review** (100%) | Role matrices for student, parent, and admin defined in `architecture.md` (§8); enforced via `StudentRoleGuard`, `ParentRoleGuard`, and `AdminRoleGuard`. | None. Permission models are fully active. |
| **P1-005** | Backend/API Foundation | P0 - Critical | **Ready for Review** (100%) | NestJS baseline, Zod env validation, error envelope, Pino logging, Terminus health probes, Swagger OpenAPI, and passing unit/integration suites. | None. Foundational service baseline is complete. |
| **P1-007** | Database & Data Model | P0 - Critical | **Ready for Review** (100%) | PostgreSQL migrations with `node-pg-migrate`, deterministic seeding, UTC timestamps, and UUID primary keys active. | None. Database baseline is complete. |
| **P1-008** | Database & Data Model | P0 - Critical | **Ready for Review** (100%) | Tables `users`, `user_profiles`, `user_roles`, and `auth_identities` supporting student, parent, and admin accounts. | None. Identity schema is complete. |
| **P1-010** | Database & Data Model | P0 - Critical | **Ready for Review** (100%) | Normalized taxonomy tables (`subjects`, `topics`, `categories`, `questions`) supporting question classification. | None. Question taxonomy model is complete. |
| **P1-011** | Database & Data Model | P0 - Critical | **Ready for Review** (100%) | Schema `question_options` and evaluation logic supporting single and multi-correct options. | None. Answer model is complete. |
| **P1-012** | Database & Data Model | P1 - High | **Ready for Review** (100%) | Question explanations table and columns supporting text explanations and media references. | None. Explanation schema is complete. |
| **P1-016** | Database & Data Model | P0 - Critical | **Ready for Review** (100%) | Schema `parent_child_links` and `parent_invites` with expiring tokens, audit trails, and status transitions. | None. Linking schema is complete. |
| **P1-017** | Authentication & Accounts | P0 - Critical | **Ready for Review** (100%) | Google OAuth authorization code exchange, ID token validation, and account linkage in `GoogleIdentityService`. | None. Google sign-in is operational. |
| **P1-018** | Authentication & Accounts | P0 - Critical | **Ready for Review** (100%) | Email OTP generation, HMAC-SHA256 hashing, 5-minute expiry, single-use consumption, and 60-second resend cooldown in `OtpService`. | Throttling applied to `request-otp` and `verify-email`, but needs verification against distributed abuse. |
| **P1-019** | Authentication & Accounts | P0 - Critical | **Ready for Review** (100%) | Refresh token rotation, SHA-256 token hashing, session revocation, and device tracking in `TokenService` and `SessionsRepository`. | Session logout and token refresh throttling are active. |
| **P1-020** | Authentication & Accounts | P0 - Critical | **Ready for Review** (100%) | Role guards and relationship authorization in `ParentLinksService` preventing unauthorized cross-tenant data access. | None. Authorization checks are verified. |
| **P1-026** | Question Bank & Content | P0 - Critical | **Ready for Review** (100%) | CSV/Excel specification and template schemas defined in `bulk-question.schemas.ts`. | None. Template structure is complete. |
| **P1-027** | Question Bank & Content | P0 - Critical | **Ready for Review** (100%) | Bulk question parser in `BulkQuestionsService` parsing rows into staging, validating taxonomy, and returning row-level error reports. | Missing rate limit protection: `@Post('bulk')` currently uses `@SkipThrottle()`. |
| **P1-032** | Practice Tests | P0 - Critical | **Ready for Review** (100%) | Practice session start, single question answering (`PUT practice/sessions/:sessionId/items/:itemId/answer`), and scoring in `PracticeService`. | Missing rate limiting and anti-automation protection on answer submission and session creation endpoints. |
| **P1-037** | Mock Exams | P0 - Critical | **Ready for Review** (100%) | Mock exam attempt start, question answering (`PUT attempts/:attemptId/items/:itemId/answer`), section locking, and completion in `MockExamsService`. | Missing rate limiting and anti-automation protection on exam answer submission and attempt finalization. |
| **P1-054** | Parent Experience | P0 - Critical | **Ready for Review** (100%) | Parent invite creation, resending, and status tracking in `ParentInvitesService`. | Missing rate limiting on invite redemption (`redeem`) and cancellation (`cancel`), leaving invite codes vulnerable to brute-force probing. |
| **P1-058** | Security & Compliance | P0 - Critical | **In Progress (Target)** | Basic in-memory `@nestjs/throttler` exists with policies for `login`, `signup`, `refresh`, `logout`, `otp`, and `link`. | 1. Rate limiting missing on Import (`@SkipThrottle` used).<br>2. Rate limiting missing on Answer Submissions (practice and mock exams).<br>3. Rate limiting missing on Invite Redemption (`redeem`) and link approvals.<br>4. No structured logging of suspicious behavior (429 events and repeated auth/probe failures are not logged with audit metadata).<br>5. Anti-automation thresholds for automated test scraping not defined in environment schema. |

---

## 3. Product and Architectural Decisions (Resolved)

All architectural and security decisions for Case P1-058 have been established as follows:

| Decision Item | Chosen Approach | Implementation Details |
|---|---|---|
| **D1: Endpoint Rate Limit Policies Matrix** | **Explicit Named Policies per Endpoint Category** | Configure granular rate limits via `@nestjs/throttler` over a 60-second window:<br>1. **General**: 100 req/min per IP (`THROTTLE_GENERAL_LIMIT`).<br>2. **Auth (Sign-in/Sign-up)**: 5 req/min per IP (`THROTTLE_LOGIN_LIMIT` / `THROTTLE_SIGNUP_LIMIT`) plus 5 req/min per email (`THROTTLE_ACCOUNT_LIMIT`).<br>3. **OTP**: 5 req/min per IP and per email (`THROTTLE_OTP_LIMIT`).<br>4. **Invites & Links**: 20 req/min per IP (`THROTTLE_LINK_LIMIT`) covering invite creation, resending, revoking, and code redemption.<br>5. **Import**: 20 req/min per admin IP (`THROTTLE_IMPORT_LIMIT`), replacing `@SkipThrottle()`.<br>6. **Answer Submissions**: 60 req/min per student (`THROTTLE_SUBMISSION_LIMIT`), allowing normal fast test-taking while preventing bot automated scraping. |
| **D2: Import Protection vs Admin Usability** | **Dedicated Import Throttler (20 req/min)** | Remove `@SkipThrottle()` from `@Post('admin/questions/bulk')`. The admin portal uploads files in chunks of 50 rows. A limit of 20 chunks per minute allows importing up to 1,000 questions per minute under normal operations while protecting the server from denial-of-service or memory exhaustion attacks. |
| **D3: Anti-Automation on Student Submissions** | **User-Centric Submission Pacing (60 req/min)** | Apply a dedicated `submission` throttler policy to `PUT .../items/:itemId/answer` and `POST .../submit`. Legitimate students rarely submit answers faster than 1 question per second. A threshold of 60 answers per minute (1 per second sustained) provides ample headroom for fast students skipping or clicking through questions while stopping automated headless scrapers from draining the entire question bank. |
| **D4: Brute-Force Protection for Invite Codes** | **Extend Link Throttler to Redemption** | Add `@AuthThrottle('link')` to `POST /parent/invites/redeem`, `POST /student/parent-invites/redeem`, and student approval routes. This limits code guessing attempts to 20 per minute per IP, preventing automated brute-force attacks against short invite codes. |
| **D5: Suspicious Behavior Structured Logging** | **Dedicated Security Audit Logger in Exception Filter** | When a request triggers `ThrottlerException` (HTTP 429), `GlobalExceptionFilter` logs a structured warning containing: `eventType: 'RATE_LIMIT_EXCEEDED'`, client IP, route path, HTTP method, userId (if authenticated), user agent, requestId, and the calculated `Retry-After` seconds. In addition, repeated failed authentication or OTP attempts emit `SECURITY_SUSPICIOUS_ACTIVITY` logs with redacted parameters. |
| **D6: Storage Strategy for Rate Limit Counters** | **In-Memory Baseline with Shared Store Readiness** | Continue using process in-memory counters for current single-instance deployment, with configuration isolated in `src/common/guards/throttler.options.ts` so switching to Redis requires only passing the storage adapter without modifying route decorators or controllers. |

---

## 4. Sequential Execution Pipeline

The execution of Case P1-058 is structured into three sequential phases to ensure full testability and zero regression across existing services:

```
+----------------------------------------------------------------------------------------------------+
|                                    STEP-BY-STEP EXECUTION FLOW                                     |
+------------------------------------+----------------------------------+----------------------------+
| Phase 1: Throttler Infrastructure  | Phase 2: Endpoint Coverage & Logs| Phase 3: Tests & Verification |
| (Policies, Config, and Schemas)    | (Controllers & Exception Filter) | (Unit, Integration, Audit) |
+------------------------------------+----------------------------------+----------------------------+
| 1. Expand AuthThrottlePolicy to    | 1. Add @AuthThrottle('import')   | 1. Unit tests for throttler|
|    include 'import', 'submission'. |    to admin question bulk upload.|    options and policies.   |
| 2. Add THROTTLE_IMPORT_LIMIT and   | 2. Add @AuthThrottle('submission')| 2. Unit tests for security |
|    THROTTLE_SUBMISSION_LIMIT to    |    to practice & exam answers.   |    audit logging on 429.   |
|    env.schema.ts & security.config.| 3. Add @AuthThrottle('link') to  | 3. Integration tests for   |
| 3. Configure named throttlers and  |    all invite redeem endpoints.  |    import, invite, and     |
|    trackers in throttler.options.ts| 4. Add structured security logs  |    answer rate limiting.   |
|                                    |    for 429s in exception filter. | 4. Add rate-limit check to |
|                                    | 5. Add audit logs on failed OTP. |    verify-security.ts CLI. |
| -> Target: Configuration Ready     | -> Target: Full Endpoint Shield  | -> Target: 100% Pass Rate  |
+------------------------------------+----------------------------------+----------------------------+
```

---

## 5. Detailed Breakdown of the Dependency Chain Cases

### 5.1 Case P1-018: Email OTP Authentication
- **Role in Chain**: Generates, delivers, and verifies 6-digit one-time passcodes for passwordless login.
- **Abuse Vulnerability**: Without strict per-IP and per-account rate limits, attackers can use OTP endpoints to perform SMS/email spamming (cost inflation), phone/email enumeration, or automated brute-force guessing of the 6-digit code.
- **Status & Verification**:
  - `POST /api/v1/auth/otp/send`: Throttled by `otp` (5 req/min per IP) and `account` (5 req/min per email).
  - `POST /api/v1/auth/otp/verify`: Throttled by `otp` and `account`.
  - `POST /api/v1/auth/otp/resend`: Throttled by `otp` and `account` with 60-second database cooldown.
  - Acceptance Criteria: Validated. Brute-force and spam attempts are throttled with HTTP 429.

### 5.2 Case P1-054: Parent-Child Invite Workflow
- **Role in Chain**: Allows parents and students to initiate, accept, and redeem link invitations using 6-character codes.
- **Abuse Vulnerability**: If invite code redemption endpoints are unthrottled, an attacker can brute-force 6-character codes to link unauthorized accounts or enumerate valid invitations.
- **Status & Verification**:
  - `POST /api/v1/parent/invites` and `POST .../resend`: Covered by `@AuthThrottle('link')`.
  - `POST /api/v1/parent/invites/redeem`: Currently missing `@AuthThrottle('link')` - must be protected.
  - `POST /api/v1/student/parent-invites/redeem`: Currently missing `@AuthThrottle('link')` - must be protected.
  - `POST /api/v1/student/parent-requests/:linkId/approve`: Currently missing `@AuthThrottle('link')` - must be protected.

### 5.3 Case P1-027: Question Import Parser and Staged Validation
- **Role in Chain**: Parses CSV/Excel question batches in chunks of up to 50 rows, validating taxonomy, options, and media references.
- **Abuse Vulnerability**: Question import involves heavy parsing, schema validation, and database writes. Unthrottled access allows a malicious or misconfigured admin client to flood the server, causing thread pool starvation and memory exhaustion.
- **Status & Verification**:
  - `POST /api/v1/admin/questions/bulk`: Currently has `@SkipThrottle()`. Must be replaced with `@AuthThrottle('import')` with a safe threshold (`THROTTLE_IMPORT_LIMIT=20`).

### 5.4 Case P1-032 & P1-037: Practice and Mock Exam Answer Submissions
- **Role in Chain**: Records student answers to interactive questions, evaluates correctness, updates stats, and finalizes attempts.
- **Abuse Vulnerability**: Headless bots can rapidly answer questions in automated loops to scrape question content, reverse-engineer answer keys, or distort platform question psychometrics.
- **Status & Verification**:
  - `PUT /api/v1/student/practice/sessions/:sessionId/items/:itemId/answer`: Currently unthrottled. Must be protected by `@AuthThrottle('submission')`.
  - `POST /api/v1/student/practice/sessions/:sessionId/submit`: Currently unthrottled. Must be protected by `@AuthThrottle('submission')`.
  - `PUT /api/v1/student/mock-exams/attempts/:attemptId/items/:itemId/answer`: Currently unthrottled. Must be protected by `@AuthThrottle('submission')`.
  - `POST /api/v1/student/mock-exams/attempts/:attemptId/submit`: Currently unthrottled. Must be protected by `@AuthThrottle('submission')`.

---

## 6. Step-by-Step Implementation Plan for Case P1-058

### Stage 1: Rate Limiting & Throttler Architecture Expansion
1. **Extend Throttle Policies**:
   Update `src/common/decorators/auth-throttle.decorator.ts`:
   - Expand `AuthThrottlePolicy` type to include `'import'` and `'submission'`.
   - Maintain `ACCOUNT_THROTTLED_POLICIES` for account-keyed limits (`signup`, `login`, `otp`).
2. **Environment Configuration**:
   Update `src/config/env.schema.ts` and `src/config/security.config.ts`:
   - Add `THROTTLE_IMPORT_LIMIT: positiveInt.default(20)` (20 chunks/minute per IP).
   - Add `THROTTLE_SUBMISSION_LIMIT: positiveInt.default(60)` (60 answers/minute per student/IP).
   - Add to `SecurityConfig.throttle` interface.
3. **Register Named Throttlers**:
   Update `src/common/guards/throttler.options.ts`:
   - Register route policy throttler for `'import'`: `limit: throttle.importLimit`.
   - Register route policy throttler for `'submission'`: `limit: throttle.submissionLimit`.
   - Add a student/user tracker for submissions: track by `userId` when authenticated, falling back to client IP.

### Stage 2: Endpoint Coverage & Protection
1. **Admin Bulk Question Import**:
   In `src/modules/questions/questions.controller.ts`:
   - Remove `@SkipThrottle()` from `@Post('bulk')`.
   - Apply `@AuthThrottle('import')`.
2. **Parent and Student Invite Redemption**:
   In `src/modules/parent/parent-invites.controller.ts`:
   - Add `@AuthThrottle('link')` to `POST :id/redeem`, `DELETE :id`.
   In `src/modules/parent/student-invites.controller.ts`:
   - Add `@AuthThrottle('link')` to `POST :id/redeem`, `DELETE :id`.
   In `src/modules/parent/parent-requests.controller.ts`:
   - Add `@AuthThrottle('link')` to `POST :linkId/approve`, `POST :linkId/revoke`.
3. **Student Practice Answer Submissions**:
   In `src/modules/practice/practice.controller.ts`:
   - Add `@AuthThrottle('submission')` to `PUT practice/sessions/:sessionId/items/:itemId/answer`.
   - Add `@AuthThrottle('submission')` to `POST practice/sessions/:sessionId/submit`.
4. **Student Mock Exam Answer Submissions**:
   In `src/modules/mock-exams/mock-exams.controller.ts`:
   - Add `@AuthThrottle('submission')` to `PUT attempts/:attemptId/items/:itemId/answer`.
   - Add `@AuthThrottle('submission')` to `POST attempts/:attemptId/sections/:sectionId/submit`.
   - Add `@AuthThrottle('submission')` to `POST attempts/:attemptId/submit`.

### Stage 3: Suspicious Behavior Structured Logging
1. **Log 429 Throttle Events**:
   Update `src/common/filters/global-exception.filter.ts`:
   - Inside `if (exception instanceof ThrottlerException)`, capture request details: client IP, normalized URL, method, authenticated user ID (from `request.user`), user-agent, and retry-after header.
   - Emit a structured warning:
     `this.logger.warn({ event: 'SECURITY_RATE_LIMIT_EXCEEDED', ip, method: request.method, path: request.url, userId, retryAfter }, 'Rate limit exceeded: throttling request')`.
2. **Log Repeated Authentication Failures**:
   In `src/modules/auth/auth.service.ts`:
   - When OTP verification fails or password verification fails, emit a structured audit log:
     `this.logger.warn({ event: 'SECURITY_AUTH_FAILED', ip, email: maskEmail(email), reason }, 'Authentication attempt failed')`.
3. **Log Invalid Invite Code Probes**:
   In `src/modules/parent/parent-invites.service.ts`:
   - When an invite code is not found or already consumed during redemption, emit a structured audit log:
     `this.logger.warn({ event: 'SECURITY_INVITE_PROBE_FAILED', ip, userId }, 'Invalid invite code redemption attempt')`.

### Stage 4: Automated Verification Script
1. Update `src/scripts/verify-security.ts`:
   - Add Check 8 (`verifyRateLimitingCoverage`): Assert that all 5 endpoint categories (OTP, login, invites, import, submissions) have active throttler configurations and that `env.THROTTLE_IMPORT_LIMIT` and `THROTTLE_SUBMISSION_LIMIT` are valid integers.
   - Run `npm run verify:security` and ensure 8/8 checks pass cleanly.

### Stage 5: Test Suite Additions
1. **Unit Tests**:
   - `test/unit/throttler.spec.ts`: Verify that `buildThrottlerOptions` configures all named throttlers (`default`, `signup`, `login`, `refresh`, `logout`, `otp`, `link`, `import`, `submission`, `account`).
   - `test/unit/global-exception.filter.spec.ts`: Verify that `ThrottlerException` emits a structured `SECURITY_RATE_LIMIT_EXCEEDED` warning log and sets the `Retry-After` header.
2. **Integration Tests**:
   - Update `test/integration/rate-limit.int-spec.ts`:
     - Test bulk question import throttling (exceeding `THROTTLE_IMPORT_LIMIT` yields 429).
     - Test invite redemption throttling (exceeding `THROTTLE_LINK_LIMIT` yields 429).
     - Test answer submission throttling (exceeding `THROTTLE_SUBMISSION_LIMIT` yields 429).
     - Verify that normal-paced student submissions and health probes are never throttled.

---

## 7. Implementation Impact on Other Repositories

### 7.1 Student and Parent Mobile App (`D:\lm` - React Native / Expo)
- **429 Handling**: The mobile API client must detect HTTP 429 `RATE_LIMIT_EXCEEDED` and inspect the `Retry-After` response header.
- **User Feedback**:
  - On OTP requests: display a countdown timer ("Please wait X seconds before requesting another code").
  - On sign-in: show a clear warning message ("Too many attempts. Please try again in X seconds").
  - On practice/exam answer submissions: if a rapid series of answers hits the limit, buffer the answer locally and retry after the specified window, preventing student answer loss.

### 7.2 Admin Web Portal (`learning-app-admin` - Vite + React)
- **Bulk Question Import**:
  - The admin bulk import workflow reads Excel/CSV files and uploads rows in chunks of 50.
  - With `THROTTLE_IMPORT_LIMIT=20` (20 chunks/min = 1,000 questions/min), normal imports proceed smoothly.
  - If a file exceeds 1,000 questions (more than 20 chunks), the admin client should introduce a small inter-chunk delay (e.g. 250 ms) or automatically catch 429, wait for `Retry-After`, and resume chunk uploads without failing the batch.

### 7.3 Student / Parent Web App (`learning-web-app` - Next.js)
- **Client Parity**:
  - Standard error interceptor catches 429 responses and displays a non-intrusive toast notification indicating when the rate limit window resets.
