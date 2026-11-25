# 2026-11-24 - secure-data-protection (backend)

## Summary

This PR implements backend data protection, encryption baselines, privacy controls, and automated retention cleanup (Phase 1 cases P1-004, P1-005, and P1-057). It adds production database TLS enforcement, weak secret rejection, HSTS preload headers, reverse-proxy HTTPS enforcement, logger PII redaction for phone numbers, emails, and OTP tokens, an automated background retention purge service using PostgreSQL advisory locks, immediate user account deletion with data anonymization (`DELETE /api/v1/auth/account`), account data export (`GET /api/v1/auth/account/export`), an automated security verification CLI (`npm run verify:security`), and formal NFR and data protection architecture documentation.

All existing student authentication, admin workflows, exam blueprints, question uploads, and practice sessions continue working without disruption. Checked live: all 199 Vitest unit tests pass, TypeScript compilation passes with zero errors, Prettier check passes, and all 7 automated security verification checks pass.

**External production secret injection (AWS Secrets Manager / Vault) is not included on purpose** - see "Decisions made" below. This was a choice, not a miss.

## What changed

| File | Change |
|---|---|
| `architecture.md` | Added Section 26 (Non-Functional Requirements baseline covering 50k MAU scale, p95 latency targets, RPO/RTO SLAs, and audit trails) and Section 27 (Secure Data Protection baseline covering AWS RDS KMS CMK encryption, database TLS, zero phone logging, secret rotation runbooks, and retention intervals). Updated Section 1 summary table. |
| `package.json` | Added `verify:security` npm script running `node --env-file-if-exists=.env dist/scripts/verify-security.js`. |
| `src/app.module.ts` | Registered `MaintenanceModule` in application imports. |
| `src/config/env.schema.ts` | Added `ALLOW_INSECURE_INTERNAL_DB` option. Enforced `DB_SSL=true` in production unless override is set for VPC peering. Added validation rejecting weak, repetitive, or placeholder values for `JWT_ACCESS_SECRET` and `OTP_HMAC_SECRET` in production. |
| `src/modules/auth/auth.controller.ts` | Added `DELETE /api/v1/auth/account` for immediate account deletion and `@Get(['account/export', 'export'])` for user data export, guarded by `AccessTokenGuard` and documented with OpenAPI metadata. |
| `src/modules/auth/auth.service.ts` | Added `deleteAccount` (runs anonymization transaction) and `exportAccountData` service methods with audit logging. |
| `src/modules/auth/repositories/users.repository.ts` | Implemented `deleteAccountAndAnonymize` (wipes profile, sets pseudonym email, disables account, revokes sessions, removes bookmarks, and unlinks parent links) and `exportUserData` (compiles profile, attempt counts, and bookmarks). |
| `src/modules/maintenance/` (new) | Created `MaintenanceModule`, `RetentionPurgeService`, and `RetentionPurgeRepository` running hourly background purges under PostgreSQL session advisory lock `pg_try_advisory_xact_lock(hashtext('retention-purge-jobs'))`. |
| `src/shared/http/configure-http.ts` | Configured Helmet with explicit HSTS preload (`maxAge: 31536000`, `includeSubDomains: true`, `preload: true`) and added production reverse-proxy HTTPS redirect middleware. |
| `src/shared/logger/logger.options.ts` | Exported and expanded `REDACT_PATHS` to redact `phone`, `email`, `otp`, `code`, and variants from logs, request bodies, and error envelopes. |
| `src/scripts/verify-security.ts` (new) | Created standalone CLI script running 7 security verification checks across secret strength, Argon2id parameters, database TLS, CORS HTTPS, logger redaction coverage, and payload caps. |
| `test/unit/account-deletion.spec.ts` (new) | Unit test suite verifying account anonymization queries, active session revocations, and data export formatting. |
| `test/unit/retention-purge.spec.ts` (new) | Unit test suite verifying advisory lock acquisition, transaction handling, and execution across all 4 purge types. |
| `test/unit/security-verification.spec.ts` (new) | Unit test suite verifying the 7 automated security verification checks and logger redaction coverage. |
| `test/unit/env.schema.spec.ts` | Added test cases asserting production rejection of weak secrets and missing database SSL. |
| `src/modules/exams/schemas/exam-request.schemas.ts`, `src/modules/questions/schemas/question-request.schemas.ts` | Code formatting alignment via Prettier (no functional changes). |

## Why

The platform handles student test data and user identities that require production-grade data protection, compliance, and privacy controls:
1. Satisfying NFR baseline targets for Phase 1 (case P1-004) and verifying foundational service health (case P1-005).
2. Implementing data protection safeguards for case P1-057: transport encryption, database encryption baselines, zero PII exposure, and automated data minimization.
3. Enabling student Right to Erasure (Apple App Store Guideline 5.1.1(v)) and Right to Portability (GDPR/privacy compliance).

Setting up the Zod schema checks was straightforward; the primary engineering effort was coordinating transactional account anonymization without violating foreign key constraints on historical assessment attempts and implementing safe multi-instance background retention purges.

## How

**Database TLS enforcement**: In production, `envSchema` rejects configurations where `DB_SSL` is false unless `ALLOW_INSECURE_INTERNAL_DB=true` is explicitly set for internal VPC peering. Connections to PostgreSQL enforce TLS certificate validation.

**Secret entropy validation**: In production, `JWT_ACCESS_SECRET` and `OTP_HMAC_SECRET` must be at least 32 characters and cannot match repetitive characters or common placeholder patterns (`secret`, `placeholder`, `changeme`, `example`).

**Transport security**: Helmet sets strict HSTS headers with `preload` enabled and a 1-year max age. Production middleware inspects `x-forwarded-proto` and returns HTTP 301 redirects to `https://` for unencrypted requests.

**Zero phone and PII log redaction**: The structured Pino logger redacts `phone`, `email`, `otp`, `code`, passwords, and tokens across top-level fields, nested objects, and request bodies before log output is emitted.

**Universal background retention purges**: `RetentionPurgeService` runs on an hourly schedule using PostgreSQL transaction advisory locks (`pg_try_advisory_xact_lock`), ensuring only one container instance executes cleanup at a time. It purges auth sessions expired or revoked for more than 30 days, email OTPs older than 7 days, parent invites older than 30 days, and unconfirmed media assets older than 24 hours.

**Right to Erasure with question calibration preservation**: When `DELETE /api/v1/auth/account` is invoked, `UsersRepository.deleteAccountAndAnonymize` transactionally wipes personal profile details, sets `email` to `deleted-<uuid>@prepify.local`, revokes active sessions and refresh tokens, unlinks parent links, and deletes question bookmarks. Historical rows in `assessment_attempts` are preserved with user identity severed, preventing crashes on `ON DELETE RESTRICT` constraints and protecting question psychometric calibration data.

**User data portability export**: `GET /api/v1/auth/account/export` compiles the authenticated user's account details, profile, target test, question bookmarks, and practice/exam attempt totals into a structured JSON payload.

**Automated security verification CLI**: `npm run verify:security` runs 7 automated checks against the active configuration, validating secret entropy, Argon2id parameters, database TLS, CORS security, PII redactions, and payload limits.

## Decisions made

**Anonymization over hard deletes (Decision D1).** `assessment_attempts.user_id` has an `ON DELETE RESTRICT` constraint referencing `users(id)` to prevent destroying historical test data. We anonymize user identity (`deleted-<uuid>@prepify.local`, profile wiped, credentials cleared) rather than executing `DELETE FROM users`, satisfying Right to Erasure while preserving question difficulty calibration.

**Immediate deletion with zero grace period (Decision D2).** We opted for immediate deletion and session termination upon confirmation, with no 14-day recovery window, providing a clean user privacy experience and satisfying Apple App Store Guideline 5.1.1(v).

**AWS RDS AES-256 encryption with KMS CMK (Decision D3).** Documented infrastructure baseline requiring AWS RDS storage encryption with customer-managed keys (CMK), encrypted automated backups and snapshots, app-to-db TLS, VPC private subnet isolation, and zero phone logging.

**Deferred external secret injection provider (Decision D4).** External secret injection tools (such as AWS Secrets Manager or HashiCorp Vault) were deferred because there is no requirement for production secret injection at this stage. Hardening focuses on environment validation, entropy checks, and preventing secrets from ever appearing in error traces or logs.

## Notes

- In local development (`NODE_ENV=development`), `DB_SSL=false` continues to work with local PostgreSQL.
- The retention purge background timer is disabled during unit and integration test runs (`process.env.NODE_ENV === 'test'`) to prevent hanging timers.
- Prettier formatting check and Oxlint pass across all source and test files.

## Next steps

- Mobile team implements in-app "Delete Account" button and "Export My Data" action in `D:\lm` following [`MOBILE_APP_CHANGES.md`](file:///D:/Livik-projects/dev-docs/learning-app/tasks/feat-secure-data-protection/MOBILE_APP_CHANGES.md).
- Admin web team implements PII masking for student phone and email records on user tables and adds support for displaying anonymized student accounts in analytics.

## Test plan

**Automated tests (all 199 passing):**
- [x] `test/unit/env.schema.spec.ts`: verifies rejection of weak secrets and missing DB SSL in production.
- [x] `test/unit/account-deletion.spec.ts`: verifies transactional execution of profile wiping, pseudonym email assignment, session revocation, and data export aggregation.
- [x] `test/unit/retention-purge.spec.ts`: verifies advisory locking and execution of all 4 purge queries.
- [x] `test/unit/security-verification.spec.ts`: verifies that all 7 security checks pass and redaction lists cover phone, email, and OTP tokens.
- [x] `npm run verify:security`: all 7 security checks passed (exit code 0).
- [x] `npm run typecheck`: clean compilation with 0 TypeScript errors.
- [x] `npm run build`: `nest build` completed cleanly with exit code 0.
- [x] `npm run format:check`: all files conform to Prettier code style.
- [x] `npx oxlint src/ test/`: 0 errors.

**Manual checks (local):**
- [x] Ran `npm run verify:security` and confirmed all 7 checks pass with detailed green checkmarks.
- [x] Tested `envSchema` with weak secret strings and confirmed descriptive startup errors without leaking secret values.
- [x] Verified logger redaction paths defensively redact phone numbers, email addresses, and OTP codes.
