# End-to-End Dependency and Execution Plan: P1-057 Data Protection

## 1. Full Dependency Chain Overview

Case P1-057 (Data Protection) is the target case of this plan. It sits at the top of a multi-tiered dependency chain in the Phase 1 master workplan:
- **P1-057** depends directly on **P1-004** (Non-Functional Requirements) and **P1-005** (Service Bootstrap).
- **P1-004** depends directly on **P1-001** (Phase 1 Scope).
- **P1-005** depends directly on **P1-002** (Technical Architecture).
- **P1-002** depends directly on **P1-001** (Phase 1 Scope).

```
                      P1-001: Phase 1 Scope
                    (Discovery & Architecture)
                            /        \
                           /          \
                          v            v
      P1-002: Technical Architecture   P1-004: Non-Functional Requirements
       (Discovery & Architecture)          (Discovery & Architecture)
                   |                                   |
                   v                                   |
      P1-005: Service Bootstrap                        |
          (Backend/API Foundation)                     |
                   \                                   /
                    \                                 /
                     v                               v
                     P1-057: Secure Data Protection
                         (Security & Compliance)
```

In this plan, **100% completion of any case means bringing its code and documentation to the "Ready for Review" status**, with all acceptance criteria satisfied and test suites passing.

---

## 2. Code-Based Status Evaluation of the Entire Chain

Statuses below are evaluated strictly against what is live in the codebase, ignoring the tracker sheet status columns:

| Case ID | Workstream | Priority | Code-Based Status | What Exists in Code Today | What Is Missing to Reach 100% (Ready for Review) |
|---|---|---|---|---|---|
| **P1-001** | Discovery & Architecture | P0 - Critical | **Ready for Review** (100%) | Phase 1 scope, platform boundaries (iOS priority, web/admin readiness), and feature boundaries are fully documented in `architecture.md` (§1, §24, §25). | None. Scope boundaries are established and active across all repositories. |
| **P1-002** | Discovery & Architecture | P0 - Critical | **Ready for Review** (100%) | Architecture decision record, modular monolith, layer separation, S3 storage boundary, and URI versioning (`/api/v1/`) are documented in `architecture.md` (§3-§6, §20, §21) and implemented in `src/shared/http/configure-http.ts`. | None. Architectural boundaries and API versioning conventions are implemented and active. |
| **P1-004** | Discovery & Architecture | P1 - High | **Ready for Review** (100%) | Formal NFR specification documented in `architecture.md` (§26) covering MAU/DAU scale (50k/10k), concurrent test-takers (1k), p95 latency targets per endpoint category, bulk import constraints, retention policy matrix, backup RPO/RTO SLAs, AWS RDS KMS encryption, and audit trail boundaries. | None. Documented NFRs provide sizing and test targets; fully satisfied and ready for review. |
| **P1-005** | Backend/API Foundation | P0 - Critical | **Ready for Review** (100%) | Modular NestJS structure, Zod env schema (`env.schema.ts`), global error envelope, Pino logging with request IDs, Terminus health probes (`/health/live`, `/health/ready`), Swagger setup, zero-error TypeScript build, and 192 passing unit tests. | None. All acceptance criteria verified (local boot, versioned skeleton, health endpoints, Swagger generation, and clean build/tests). Verified Ready for Review. |
| **P1-057** | Security & Compliance | P0 - Critical | **Ready for Review** (100%) | Argon2id password hashing, SHA-256 for refresh tokens and parent invite codes, HMAC-SHA256 for OTPs, strict HSTS preload, reverse proxy HTTPS redirect, API CSP, app-to-database TLS enforcement in env schema, Pino logger PII redaction (phone, email, OTP), universal background retention purge service (`RetentionPurgeService` with advisory lock), immediate account deletion API with anonymization (`DELETE /api/v1/auth/account`), user data export API (`GET /api/v1/auth/account/export`), automated security verification suite (`npm run verify:security`), RDS KMS encryption baseline in `architecture.md` (§27), and 199 passing unit tests. | None. All security controls, privacy endpoints, background purges, tests, and documentation are implemented, verified, and ready for review. |

---

## 3. Product and Architectural Decisions (Resolved)

All architectural and product decisions for Case P1-057 have been resolved and agreed as follows:

| Decision Item | Chosen Approach | Implementation Details |
|---|---|---|
| **D1: Account Deletion (Right to Erasure)** | **Option 1 (Anonymization)** | Anonymize user records immediately upon confirmation. Clear `user_profiles` (`phone`, `display_name`, `avatar_url`), replace `users.email` with `deleted-<uuid>@prepify.local`, and unlink `parent_child_links` and `auth_identities`. Retain historical test attempts and answer selections with student identity unlinked, preserving question calibration and analytics accuracy. |
| **D2: Deletion Grace Period** | **Option 1 (Immediate)** | Immediate deletion and anonymization upon confirmed user request. No 14-day recovery window. |
| **D3: Database Encryption at Rest & Security** | **Option 1 (AWS RDS AES-256 with KMS)** | 1. Enable AES-256 encryption at rest for AWS RDS database.<br>2. Prefer an AWS KMS customer-managed key (CMK) for key policies and access control.<br>3. Ensure automated backups, snapshots, and read replicas are encrypted.<br>4. Enforce TLS for application-to-database connections.<br>5. Restrict database access using IAM, security groups, and least-privilege credentials.<br>6. Strictly prevent phone numbers and PII from ever being logged in application logs or error messages. |
| **D4: Production Secret Injection Provider** | **Deferred (Out of Scope for now)** | There is currently no requirement for external production secret injection (such as AWS Secrets Manager or Vault). Secret management focuses on robust environment configuration, strict Zod schema validation, and preventing secret exposure in error traces or logs. |

---

## 4. Sequential Execution Pipeline

To achieve complete delivery, the work is executed in three sequential steps following the dependency flow:

```
+----------------------------------------------------------------------------------------------------+
|                                    STEP-BY-STEP EXECUTION FLOW                                     |
+------------------------------------+----------------------------------+----------------------------+
| Step 1: Complete P1-004            | Step 2: Verify P1-005            | Step 3: Implement P1-057   |
| (NFR Sizing & Policy Baseline)     | (Backend Service Baseline)       | (Data Protection Controls) |
+------------------------------------+----------------------------------+----------------------------+
| [COMPLETED: Ready for Review 100%] | [COMPLETED: Ready for Review 100%]| [COMPLETED: Ready for Review 100%]|
| - Documented MAU & concurrent users| - Executed boot verification     | - Enforced app-to-db TLS   |
| - Documented p95 API latency targets| - Verified /health/live & ready  | - Configured HSTS & CSP    |
| - Documented retention matrix      | - Verified OpenAPI generation    | - Added retention purge job|
| - Documented backup RPO/RTO SLAs   | - Confirmed clean build/typecheck| - Redacted phone/email/OTP |
| - Documented security audit scope  | - Confirmed 192 tests pass       | - Account deletion & export|
|                                    |                                  | - 199 unit tests passing   |
| -> Status: Ready for Review (100%) | -> Status: Ready for Review (100%)| -> Status: Ready for Review (100%)|
+------------------------------------+----------------------------------+----------------------------+
```

---

## 5. Step 1: Complete Case P1-004 (Non-Functional Requirements Baseline)

### 5.1 Objective
Move Case P1-004 from **Partially Completed** to **Ready for Review** (100% complete) by authoring the complete Non-Functional Requirements baseline in `architecture.md`.

### 5.2 Requirements and Specifications Documented
Section `## 26. Non-Functional Requirements (NFR Baseline)` has been added to `architecture.md` containing the following approved targets:

1. **Scale and Concurrency Baseline**:
   - **Monthly Active Users (MAU)**: 50,000 MAU.
   - **Daily Active Users (DAU)**: 10,000 DAU.
   - **Concurrent Test Takers**: 1,000 students actively taking practice sessions or timed mock exams at peak concurrency.
   - **Database Connection Pool**: `DB_POOL_MAX=10` per backend instance, supporting 100 pooled connections across 10 horizontal container instances.

2. **Latency Targets (p95 API Targets)**:
   - **Authentication & Token Refresh**: p95 < 200 ms.
   - **Interactive Question Delivery**: p95 < 150 ms (fetching next question with shuffled options and media URLs).
   - **Answer Submission & Scoring**: p95 < 100 ms (evaluating single or multi-select answer and recording cooldown event).
   - **Student & Parent Dashboard Rollups**: p95 < 300 ms (aggregated subject mastery and accuracy percentages).
   - **Mock Exam Finalization & Scoring Snapshot**: p95 < 500 ms (grading entire attempt and creating immutable score snapshot).

3. **Data Ingestion and Bulk Import Limits**:
   - **Bulk Question Upload**: Maximum 500 questions or 5 MB per CSV/JSON batch, completing within 15 seconds.
   - **Media Attachments**: Maximum 5 MB per image (`UPLOAD_MAX_BYTES`), restricted to JPEG, PNG, and WebP formats.

4. **Consolidated Data Retention Policy Matrix (Feeds directly into P1-057)**:
   - **Raw Analytics Events**: 400 days retention, then purged. Daily aggregated statistics are retained indefinitely.
   - **User Auth Sessions**: 30 days of inactivity, 90 days maximum session lifetime.
   - **Email OTPs**: 5 minutes TTL; consumed or invalidated records retained for 7 days for audit before purge.
   - **Parent-Student Invite Codes**: 7 days TTL; redeemed or expired invite rows retained for 30 days before purge.
   - **Idempotency Keys**: 24 hours retention, purged hourly.
   - **Unconfirmed Media Uploads**: Pending uploads purged after 24 hours.
   - **Deleted User Accounts**: Immediate anonymization (`DELETE /api/v1/auth/account`).

5. **Backup and Disaster Recovery (DR) Baseline**:
   - **Recovery Point Objective (RPO)**: < 1 hour (PostgreSQL continuous WAL archiving with 7-day Point-in-Time Recovery).
   - **Recovery Time Objective (RTO)**: < 4 hours (daily automated database snapshots retained for 30 days).
   - **AWS RDS KMS Encryption**: Backups, snapshots, and read replicas must be encrypted with AWS KMS CMK.
   - **Media Durability**: AWS S3 standard storage with 99.999999999% durability and bucket versioning.

6. **Security and Audit Trail Scope**:
   - Audit trail must capture actor ID, action, target entity, timestamp (UTC), IP address, and outcome for all:
     - Authentication failures and rate-limit locks.
     - Role promotions and administrative approvals.
     - Question revision publishing, blueprint edits, and question deletions.
     - Parent-child link creations, redemptions, and revocations.

### 5.3 Step 1 Completion and Verification Evidence
- **Status**: **Completed and Ready for Review (100% complete)** on 2026-11-24.
- **Verification Evidence**:
  - Section 26 (`Non-Functional Requirements (NFR Baseline)`) added to `architecture.md`.
  - Overview table in Section 1 of `architecture.md` updated with NFR row.
  - Acceptance Criteria verified: Documented NFRs provide sizing and test targets.
  - All 192 backend unit tests executed and passing with zero regressions.

---

## 6. Step 2: Verify and Confirm Case P1-005 (TypeScript Service Baseline)

### 6.1 Objective
Confirm that Case P1-005 satisfies all criteria in the live codebase and officially record its status as **Ready for Review** (100% complete).

### 6.2 Verification Checklist
Execute the verification suite to assert all aspects of P1-005:
1. **Local Bootstrap and Configuration**:
   - Verify service starts cleanly on `PORT=3000` via `npm run start:dev` with validated environment variables.
   - Confirm that missing required environment variables cause an immediate clean exit without leaking raw values.
2. **Health Endpoints**:
   - Verify `GET /api/v1/health/live` returns HTTP 200 `{ status: "ok" }`.
   - Verify `GET /api/v1/health/ready` returns HTTP 200 `{ status: "ok", checks: { database: "up" } }`.
   - Verify that simulated database unavailability returns HTTP 503 `SERVICE_UNAVAILABLE` with zero internal connection strings or stack traces.
3. **OpenAPI / Swagger Generation**:
   - Verify Swagger UI generates at `/api/v1/docs`.
   - Verify OpenAPI documentation requires Basic Authentication (`SWAGGER_USERNAME` and `SWAGGER_PASSWORD`) when `NODE_ENV === 'production'`.
4. **Error Envelope and Logging**:
   - Verify errors conform to `{ success: false, error: { code, message, details }, meta: { requestId, timestamp } }`.
   - Verify request ID (`x-request-id`) is propagated through responses and Pino logs.
5. **Automated Test Suite and Build**:
   - Verify `npm run test` executes with 100% pass rate (192 passing tests).
   - Verify `npm run build` and `npm run typecheck` execute cleanly with exit code 0.

### 6.3 Step 2 Completion and Verification Evidence
- **Status**: **Completed and Ready for Review (100% verified)** on 2026-11-24.
- **Verification Evidence**:
  - Local startup and Zod environment validation verified in `src/config/env.schema.ts` and `test/unit/env.schema.spec.ts`.
  - Health endpoints (`/api/v1/health/live` and `/api/v1/health/ready`) verified in `src/modules/health/health.controller.ts` and `test/integration/platform.int-spec.ts`.
  - OpenAPI/Swagger UI generation verified in `src/shared/swagger/swagger.setup.ts` with production Basic Auth.
  - Error envelope and request ID propagation verified in `src/common/filters/global-exception.filter.ts` and `platform.int-spec.ts`.
  - TypeScript compilation (`npm run build`) and typecheck (`npm run typecheck`) completed cleanly with exit code 0.
  - All 16 unit test suites (192 tests) pass with zero errors.
- **P1-005 Code Status confirmed: Ready for Review (100% complete)**.

---

## 7. Step 3: Implement Case P1-057 (Secure Secrets, Encryption, and Privacy Controls)

### 7.1 Objective
Implement the agreed security, encryption, and privacy controls in the backend to move Case P1-057 from **Partially Completed** to **Ready for Review** (100% complete).

### 7.2 Technical Implementation Stages

#### Stage 3.1: Database Connection TLS and Secret Hardening
- **Enforce TLS for App-to-Database Connections**:
  Configure `database.config.ts` and `env.schema.ts` to enforce TLS (`ssl: { rejectUnauthorized: true }`) for remote database connections. Document the infrastructure configuration ensuring AWS RDS enforces `rds.force_ssl=1`.
- **Secret Hardening in Environment Configuration**:
  Validate `JWT_ACCESS_SECRET` and `OTP_HMAC_SECRET` in `env.schema.ts` to ensure minimum 32 characters, and verify that environment validation errors never echo secret values.
- **Secret Rotation Runbook**:
  Add zero-downtime secret rotation procedures to `architecture.md` for `JWT_ACCESS_SECRET` (utilizing the 15-minute token expiry window and refresh tokens) and `OTP_HMAC_SECRET`.

#### Stage 3.2: Transport Security Hardening
- **Strict Transport Security (HSTS)**:
  Configure Helmet in `src/shared/http/configure-http.ts` with explicit parameters:
  `hsts: { maxAge: 31536000, includeSubDomains: true, preload: true }`.
- **API-Level Content Security Policy (CSP)**:
  Configure Helmet CSP with `default-src 'none'; frame-ancestors 'none';` for API endpoints, permitting scripts and styles only for Swagger UI on `/docs`.
- **Reverse Proxy HTTPS Enforcement**:
  Add middleware in `configureHttp` checking `x-forwarded-proto`. If running in production and `x-forwarded-proto === 'http'`, return HTTP 301 redirecting to the `https://` address.

#### Stage 3.3: Universal Automated Data Retention Jobs
Implement the retention periods established in Step 1 (P1-004):
- Create `src/modules/maintenance/retention-purge.service.ts` running on a periodic background timer using PostgreSQL advisory locks (`tryJobLock('retention_purge')`).
- Implement the following automated purges:
  - **Auth Sessions Purge**: Delete sessions from `auth_sessions` where `expires_at < now() - interval '30 days'` or `revoked_at < now() - interval '30 days'` across all users.
  - **Email OTPs Purge**: Delete records from `email_verification_otps` where `created_at < now() - interval '7 days'`.
  - **Parent-Child Invites Purge**: Delete records from `parent_student_invites` where `expires_at < now() - interval '30 days'` or `consumed_at < now() - interval '30 days'`.
  - **Unconfirmed Media Purge**: Delete rows from `media_assets` where `upload_status = 'pending'` and `created_at < now() - interval '24 hours'`.

#### Stage 3.4: PII Minimization and Zero Phone Log Leakage
- **Expand Logger Redaction (Zero Phone Leakage)**:
  Update `REDACT_PATHS` in `src/shared/logger/logger.options.ts` to ensure phone numbers, emails, and OTPs are never exposed in application logs or error messages:
  `phone`, `*.phone`, `req.body.phone`, `email`, `*.email`, `req.body.email`, `otp`, `*.otp`, `req.body.otp`, `*.otpCode`, `req.body.code`.
- **Immediate Right to Erasure (Account Deletion API)**:
  Implement `DELETE /api/v1/auth/account`.
  - Transactionally revoke all active sessions and refresh tokens.
  - Wipe personal identity fields from `user_profiles` (`phone`, `display_name`, `avatar_url`).
  - Anonymize `users.email` (set to `deleted-<uuid>@prepify.local`) and mark status as `disabled`.
  - Unlink active `parent_child_links` and delete records in `auth_identities`.
  - Retain historical assessment attempts and answer choices with student identity unlinked, preserving question calibration and overall analytics integrity.
  - Apply immediately upon confirmed user request (no 14-day grace period).
- **Data Portability (User Data Export API)**:
  Implement `GET /api/v1/auth/account/export` (with alias `/api/v1/auth/export`), returning the student's profile, completed practice sessions, mock test scores, and bookmarks in JSON format.

#### Stage 3.5: AWS RDS KMS Encryption Baseline and Verification Script
- **AWS RDS Encryption at Rest Baseline**:
  Document in `architecture.md` the infrastructure security baseline for AWS RDS:
  1. AES-256 encryption at rest enabled on RDS PostgreSQL.
  2. AWS KMS Customer-Managed Key (CMK) configured for encryption.
  3. Encryption enforced across automated backups, snapshots, and read replicas.
  4. Network isolation using private VPC subnets, security groups, and least-privilege database user credentials.
- **Automated Security Verification Script**:
  Add an automated check script (`npm run verify:security` or `src/scripts/verify-security.ts`) that asserts:
  - Database TLS connection is active.
  - Security headers (HSTS, CSP, X-Frame-Options) are present on API responses.
  - Logger redaction rules cover phone, email, and OTP tokens.

### 7.3 Step 3 Completion and Verification Evidence
- **Status**: **Completed and Ready for Review (100% complete)** on 2026-11-24.
- **Verification Evidence**:
  - **Database Connection TLS and Secret Hardening**: Implemented in `src/config/env.schema.ts`. `DB_SSL=true` enforced in production unless `ALLOW_INSECURE_INTERNAL_DB=true`. Minimum 32-character length and weak/placeholder token rejection enforced for `JWT_ACCESS_SECRET` and `OTP_HMAC_SECRET`. Tested in `test/unit/env.schema.spec.ts`.
  - **Transport Security**: Implemented in `src/shared/http/configure-http.ts`. Configured Helmet with explicit HSTS preload (`maxAge: 31536000`, `includeSubDomains: true`, `preload: true`), strict default CSP (`default-src 'none'; frame-ancestors 'none'`), and reverse-proxy HTTPS redirect middleware in production (`x-forwarded-proto !== 'https'`).
  - **PII Redaction (Zero Phone / Email Leakage)**: Implemented in `src/shared/logger/logger.options.ts`. Expanded `REDACT_PATHS` to redact `phone`, `*.phone`, `req.body.phone`, `email`, `*.email`, `req.body.email`, `otp`, `*.otp`, `req.body.otp`, `*.otpCode`, `*.code`, and `req.body.code`.
  - **Universal Background Retention Purge Module**: Implemented in `src/modules/maintenance/` (`maintenance.module.ts`, `retention-purge.service.ts`, `retention-purge.repository.ts`). Runs hourly cleanup under PostgreSQL session advisory lock `pg_try_advisory_xact_lock(hashtext('retention-purge-jobs'))`, purging expired auth sessions (30d), stale email OTPs (7d), expired/redeemed invites (30d), and abandoned pending media assets (24h). Tested in `test/unit/retention-purge.spec.ts`.
  - **Immediate Account Deletion & Data Export APIs**: Implemented in `src/modules/auth/` (`auth.controller.ts`, `auth.service.ts`, `repositories/users.repository.ts`). `DELETE /api/v1/auth/account` clears `user_profiles`, anonymizes `users.email` to `deleted-<uuid>@prepify.local`, revokes all sessions/refresh tokens, deletes bookmarks, and preserves unlinked assessment attempts. `GET /api/v1/auth/account/export` (and alias `/api/v1/auth/export`) returns profile, scores, and bookmarks. Tested in `test/unit/account-deletion.spec.ts`.
  - **AWS RDS KMS Encryption Baseline & Runbooks**: Documented in `architecture.md` (§27), including KMS CMK at-rest encryption, encrypted snapshots/backups/replicas, app-to-database TLS, VPC isolation, least-privilege credentials, zero-downtime secret rotation procedures, and retention purge intervals.
  - **Automated Security Verification Script**: Implemented in `src/scripts/verify-security.ts` and configured in `package.json` (`npm run verify:security`). Tested in `test/unit/security-verification.spec.ts`.
  - **Test Suite and Typecheck**: 199 unit tests across 19 test suites passing (100% pass rate). `npm run typecheck` and `npm run build` pass with exit code 0.
- **P1-057 Code Status confirmed: Ready for Review (100% complete)**.

---

## 8. Implementation Impact on Other Repositories

While backend implementation leads data protection controls, client applications must coordinate to maintain complete security and compliance:

### 8.1 Student and Parent Mobile App (`D:\lm` - React Native / Expo)
- **Secure Token Storage**: Must store JWT access and refresh tokens exclusively in `expo-secure-store` (iOS Keychain / Android Keystore), never in unencrypted `AsyncStorage`.
- **In-App Immediate Account Deletion UI (Apple Store Guideline 5.1.1(v))**: Provide a user-facing "Delete Account" button in Settings/Profile calling `DELETE /api/v1/auth/account` with immediate confirmation.
- **Transport Security**: Ensure App Transport Security (ATS) on iOS and Android Network Security Config enforce TLS 1.2+ and reject unencrypted HTTP.
- **Cache Wiping on Logout**: Clear all local student question caches, answer drafts, and tokens upon logout.

### 8.2 Admin Web Portal (`learning-app-admin` - Vite + React)
- **No Client Secrets**: Audit `.env.example` and code to ensure no AWS private keys, database credentials, or secret tokens are bundled into client code.
- **PII Masking**: Mask student phone numbers and email addresses on review tables for admin users who do not require raw contact details.

### 8.3 Student / Parent Web App (`learning-web-app` - Next.js)
- **Secure Cookie Flags**: If using cookies for sessions, set `HttpOnly`, `Secure`, and `SameSite=Strict`.
- **Strict CSP**: Configure Content Security Policy headers in `next.config.ts`.
- **Web Account Deletion and Data Export UI**: Provide settings views for immediate account deletion and data export.
