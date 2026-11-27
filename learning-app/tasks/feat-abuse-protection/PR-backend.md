# 2026-11-24 - abuse-protection (backend)

## Summary

This PR implements backend abuse protection, rate limiting across all sensitive and resource-intensive endpoints, anti-automation controls for student test submissions, and structured security audit logging (Phase 1 case P1-058).

Prior to this change, bulk question imports were unthrottled (`@SkipThrottle()`), student practice and mock exam answer submissions had no anti-automation pacing, and invite code redemption routes lacked rate limits. Furthermore, when requests exceeded rate limits or failed authentication, no structured security audit events were logged.

This PR adds dedicated named throttlers for bulk question imports (20 chunks/min per admin IP) and student answer submissions (60 answers/min per student/IP), extends link throttling to invite code redemptions and cancellations, adds structured security audit logs for rate limit events (HTTP 429), failed OTP verification attempts, wrong password attempts, and invalid invite code probes, and adds an automated abuse protection check to the security verification CLI (`npm run verify:security`).

All existing student authentication, admin panel operations, practice tests, mock exams, and parent linking continue working without disruption. Checked live: all 202 unit tests pass across 20 test files, TypeScript compiles with zero errors, Prettier formatting check passes with zero warnings, and all 8 security verification checks pass cleanly.

## What changed

| File | Change |
|---|---|
| `.env.example` | Added `THROTTLE_IMPORT_LIMIT=20` and `THROTTLE_SUBMISSION_LIMIT=60` under the rate limiting configuration section. |
| `src/common/decorators/auth-throttle.decorator.ts` | Expanded `AuthThrottlePolicy` type to include `'import'` and `'submission'` named policies. |
| `src/common/filters/global-exception.filter.ts` | Added structured `SECURITY_RATE_LIMIT_EXCEEDED` warning logging on `ThrottlerException` (HTTP 429) capturing IP, path, method, userId, user-agent, and `Retry-After`. Made retry-after extraction resilient to mock response objects. |
| `src/common/guards/throttler.options.ts` | Implemented `submissionTracker` (tracks students by authenticated `userId` with fallback to client IP) and registered named throttlers for `import` and `submission`. |
| `src/config/env.schema.ts` | Added `THROTTLE_IMPORT_LIMIT` (default 20) and `THROTTLE_SUBMISSION_LIMIT` (default 60) with positive integer validation. |
| `src/config/security.config.ts` | Added `importLimit` and `submissionLimit` to the security configuration throttle mapping. |
| `src/modules/auth/auth.service.ts` | Added structured `SECURITY_AUTH_OTP_FAILED` warning logs on invalid OTP submissions and `SECURITY_AUTH_FAILED` warning logs on password mismatches or unknown accounts without leaking sensitive input. |
| `src/modules/mock-exams/mock-exams.controller.ts` | Added `@AuthThrottle('submission')` to answer recording (`PUT attempts/:attemptId/items/:itemId/answer`), section submit (`POST attempts/:attemptId/sections/:sectionId/submit`), and exam submit (`POST attempts/:attemptId/submit`). |
| `src/modules/parent/parent-invites.controller.ts` | Added `@AuthThrottle('link')` to invite code redemption (`POST redeem`) and invite cancellation (`DELETE :id`). |
| `src/modules/parent/parent-invites.service.ts` | Added structured `SECURITY_INVITE_PROBE_FAILED` warning logs on invalid or expired invite code redemption attempts. |
| `src/modules/parent/parent-requests.controller.ts` | Added `@AuthThrottle('link')` to parent link approval (`POST :linkId/approve`) and revocation (`POST :linkId/revoke`). |
| `src/modules/parent/student-invites.controller.ts` | Added `@AuthThrottle('link')` to invite cancellation (`DELETE :id`). |
| `src/modules/practice/practice.controller.ts` | Added `@AuthThrottle('submission')` to single answer recording (`PUT practice/sessions/:sessionId/items/:itemId/answer`) and session submit (`POST practice/sessions/:sessionId/submit`). |
| `src/modules/questions/questions.controller.ts` | Replaced `@SkipThrottle()` on `@Post('bulk')` with `@AuthThrottle('import')` (20 chunks/minute per admin IP). |
| `src/scripts/verify-security.ts` | Added Check 8 ("Abuse Protection & Rate Limiting Coverage") verifying active limits across all 5 endpoint categories and positive env thresholds. |
| `test/integration/rate-limit.int-spec.ts` | Added integration tests verifying rate limiting on bulk question import and practice question answer submissions. |
| `test/integration/setup-env.ts` | Added generous defaults for `THROTTLE_IMPORT_LIMIT` and `THROTTLE_SUBMISSION_LIMIT` in integration test environment. |
| `test/unit/env.schema.spec.ts` | Added assertions for `THROTTLE_IMPORT_LIMIT: 20` and `THROTTLE_SUBMISSION_LIMIT: 60` default schema values. |
| `test/unit/global-exception.filter.spec.ts` | Updated mock response object in unit test to support `getHeader` alongside `getHeaders`. |
| `test/unit/security-verification.spec.ts` | Updated unit test to assert that all 8 automated security verification checks pass. |
| `test/unit/throttler.spec.ts` (new) | Created unit test suite verifying throttler options builder, all 10 named throttler configurations, submission tracker user isolation with IP fallback, and account tracker email normalization. |

## Why

Case P1-058 requires the platform to prevent automated abuse, brute-force attacks, resource exhaustion, and scraping across sensitive workflows:
1. **Denial of Service via Bulk Import**: Question bulk import parses CSV/Excel rows, runs schema validation, and writes to database tables. Leaving this endpoint unthrottled (`@SkipThrottle()`) left the server exposed to memory and database connection exhaustion.
2. **Invite Code Brute-Forcing**: Parent-child invite codes are short 6-character strings. Leaving redemption routes unthrottled allowed automated dictionary guessing.
3. **Automated Question Scraping**: Fast headless scrapers could rapidly submit answers to extract full question banks, explanations, and answer keys.
4. **Audit and Observability**: Operational teams need visibility into brute-force attempts and anomalous traffic without leaking user PII into application logs.

## How

**Granular endpoint throttling policies**: Extended `@nestjs/throttler` configuration in `throttler.options.ts` to manage 10 distinct throttlers over a 60-second window. Each sensitive route opts into a specific named policy using `@AuthThrottle('<policy>')`:
- `otp`: 5 req/min per IP and per account.
- `login` / `signup`: 5 req/min per IP and per account.
- `link`: 20 req/min per IP covering invite creation, redemption, cancellation, and approvals.
- `import`: 20 chunks/min per admin IP (supports up to 1,000 questions/min under normal operations).
- `submission`: 60 answers/min per student.

**User-aware submission tracking**: In `throttler.options.ts`, `submissionTracker` tracks test submissions by authenticated `userId`. If a user rotates proxies or switches networks, their answer submission rate is still constrained to 60 answers/min, while IP fallback is used for unauthenticated probes.

**Structured security logging**: In `global-exception.filter.ts`, every `ThrottlerException` emits a structured JSON log with `event: 'SECURITY_RATE_LIMIT_EXCEEDED'`, client IP, route path, HTTP method, userId, user-agent, and `Retry-After` seconds. In `auth.service.ts` and `parent-invites.service.ts`, failed attempts emit structured audit events without logging raw passwords, emails, or personal data.

## Decisions made

**Replace `@SkipThrottle()` on bulk import with a 20 chunk/min limit.** Bulk question import sends chunks of 50 rows. Completely disabling throttling allowed denial-of-service risks. Setting the limit to 20 chunks/min allows legitimate admins to import up to 1,000 questions per minute while safely capping abusive floods.

**Submission limit of 60 answers per minute.** A threshold of 60 answers/min (1 answer per second sustained) provides ample headroom for fast students skimming or skipping questions during practice sessions while stopping automated headless scrapers from draining the entire question bank.

**Zero PII logging in security audit events.** When authentication or invite code verification fails, security logs record only `userId`, `reason`, `attempts`, and audience. Raw passwords and email addresses are omitted to preserve compliance with privacy standards.

## Notes

- All rate limits return standard HTTP 429 `RATE_LIMIT_EXCEEDED` responses with a `Retry-After` header.
- Health probe endpoints (`/api/v1/health/live` and `/ready`) continue to bypass throttling via `@SkipThrottle()`.
- Implementation guidance for clients is documented in `MOBILE_APP_CHANGES.md` and `ADMIN_PANEL_CHANGES.md`.

## Test plan

**Automated unit and integration tests (202 unit tests passing):**
- [x] `test/unit/throttler.spec.ts`: verifies builder configuration for all 10 throttlers, submission tracker user isolation, and account tracker email normalization.
- [x] `test/unit/global-exception.filter.spec.ts`: verifies 429 response structure, `RATE_LIMIT_EXCEEDED` error code, and `Retry-After` header.
- [x] `test/unit/env.schema.spec.ts`: verifies default limits for `THROTTLE_IMPORT_LIMIT` (20) and `THROTTLE_SUBMISSION_LIMIT` (60).
- [x] `test/unit/security-verification.spec.ts`: verifies all 8 security checks pass, including Check 8 for rate limiting coverage.
- [x] `test/integration/rate-limit.int-spec.ts`: verifies that bulk question imports and practice answer submissions receive HTTP 429 when limits are exceeded.

**Manual verification (verified live):**
- [x] Sent 6 consecutive invalid login requests to `POST /api/v1/auth/login`. Requests 1 to 5 returned `401 Unauthorized`; request 6 returned `429 Too Many Requests` with `Retry-After: 60`.
- [x] Verified structured `SECURITY_RATE_LIMIT_EXCEEDED` log printed in backend terminal with client IP and request path.
- [x] Ran `npm run verify:security` and confirmed all 8 checks passed with status "Ready for Review (100% verified)".
