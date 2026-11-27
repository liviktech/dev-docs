# 2026-11-18 - student-onboarding-flow (backend)

## Summary

The backend only had password signup and login on the old users table. This PR installs the new schema from the senior, and adds passwordless sign up and login with a six digit code sent by email, plus Google sign in that creates a real session. It also adds two Claude skills that set the rules for new endpoints and NestJS code.

Signup stores the account and the answers from the signup questions (who is signing up, test, how familiar). It sends no email. Login always goes through the emailed code, for new and returning people. I have run the type check and the 40 unit tests. I have **not** run the integration tests and I have not sent a real email, see "Test plan".

**The student onboarding routes are not included on purpose** - see "Next steps" below. This was a choice, not a miss.

## What changed

| File | Change |
|---|---|
| `src/database/migrations/0001_initial-schema.sql` (new) | The senior's schema, adapted for `node-pg-migrate`. Replaces the old `0001_create-users-and-auth-sessions.sql` (removed) |
| `src/database/migrations/0002_email-otp-and-auth-sessions.sql` (new) | Makes `password_hash` optional, adds `users.email_verified_at`, and the `auth_sessions`, `auth_refresh_tokens` and `email_verification_otps` tables |
| `src/database/migrations/0003_parent-role-and-phone.sql` (new) | Adds the `parent` role and `users.phone` |
| `src/database/migrations/0004_signup-test-and-experience.sql` (new) | Adds `users.target_test` and `users.test_experience` with check constraints |
| `src/modules/auth/auth.controller.ts` | New `POST /auth/register`, `/auth/request-otp`, `/auth/verify-email`, `/auth/resend-otp`. Google login now returns a session |
| `src/modules/auth/auth.service.ts` | Register, request code, resend, verify and the Google session logic. Password login now rejects accounts with no password |
| `src/modules/auth/services/otp.service.ts` (new) | Makes the code, hashes it and compares it |
| `src/modules/auth/services/google-identity.service.ts` (new) | Checks the Google ID token |
| `src/modules/auth/repositories/email-otps.repository.ts` (new) | SQL for the codes table |
| `src/modules/auth/repositories/users.repository.ts`, `sessions.repository.ts` | Use the new column names, add lookup by email, row lock, mark verified, and save the test choice into `user_exam_preferences` |
| `src/modules/auth/schemas/*`, `interfaces/auth.interfaces.ts`, `dto/auth.dto.ts` | Strict request schemas for the new routes, new response shapes, `parent` role and the new user fields |
| `src/modules/mail/*` (new) | `MailService` using Nodemailer over SMTP, and the code email template |
| `src/config/env.schema.ts`, `mail.config.ts` (new), `security.config.ts`, `src/app.module.ts` | New `MAIL_*`, `OTP_HMAC_SECRET` and `THROTTLE_OTP_LIMIT` settings |
| `src/common/decorators/auth-throttle.decorator.ts`, `src/common/guards/throttler.options.ts` | New `otp` rate limit policy |
| `src/shared/constants/http.constants.ts` | New error codes (`ACCOUNT_NOT_FOUND`, `OTP_INVALID`, `OTP_EXPIRED`, `OTP_ATTEMPTS_EXCEEDED`, `MAIL_DELIVERY_FAILED`) |
| `.env.example` | Placeholders for the new settings. Session lifetime is now 7 days, so people log in again a week after login |
| `architecture.md` | New endpoint rows and a section on registration and email code login |
| `.claude/skills/api-endpoint-conventions/SKILL.md` (new) | Admin routes must contain `/admin`, mobile routes have no audience word |
| `.claude/skills/nestjs-best-practices/SKILL.md` (new) | Structure, SQL, security and testing rules for NestJS code in this repo |
| `test/integration/email-otp.int-spec.ts`, `google-login.int-spec.ts` (new) | HTTP tests for the code flow and Google login |
| `test/unit/otp.service.spec.ts`, `google-identity.service.spec.ts` (new), `auth-request.schemas.spec.ts`, `env.schema.spec.ts` | Unit tests for the new logic |
| `test/integration/auth.int-spec.ts`, `setup-env.ts`, `test-app.ts` | Small edits for the new fields, secrets and table cleanup |

## Why

The app needed signup with a few questions and login with an emailed code only, with no password. The schema from the senior had to become the base so later features (onboarding, exams) sit on it.

The easy part was the routes. The real work was making the code safe: one live code per person, a time limit, a limit on wrong tries, and no way to get around the limits by sending many requests at once.

## How

**Register**: Creates the user and saves the test choice in one transaction. Nothing is emailed. Allowed roles are `student` and `parent`, never `admin`.

**Login code**: A six digit code is made with `crypto.randomInt`. Only an HMAC of the code is stored, and it is compared in constant time. It lasts 5 minutes, allows 5 wrong tries, and can be sent again after 60 seconds. The cooldown is checked while the user row is locked, so two quick requests cannot both send.

**Verify**: Runs in one transaction: lock the user, find the live code, check it, mark it used, mark the email verified, and start a session. Wrong tries are saved even though the request fails.

**Email**: One pooled SMTP connection, checked at start up. If the check fails it is logged and the server still starts. If a send fails, the code is cancelled and the caller gets 503.

**Routes**: Paths come out as `/api/v1/auth/*` (global prefix plus URI versioning).

## Decisions made

**Email is plain `text` with a unique index on `lower(email)`, not `citext`.** The senior's schema used `citext`, but Windows Application Control blocked its DLL on this machine. All lookups lower case and trim the email, so it behaves the same.

**The app says when an account does not exist.** `request-otp` and `resend-otp` return 404 `ACCOUNT_NOT_FOUND`, and `register` returns 409 for a used email. This lets someone find out if an email is registered. I chose it because the app needs to tell people to sign up or log in. The `otp` and per email limits slow down guessing. It is written up in `architecture.md`.

**Added new migrations instead of editing the applied one.** I once edited `0002` after it was applied, which broke register with "column phone does not exist". I restored it and moved the change to `0003`. Never edit an applied migration.

**The old password `POST /auth/signup` and `/auth/login` are kept.** Removing them was not asked for. They now reject accounts with no password.

## Notes

- `0004` must be applied with `npm run migrate:up` on the dev database and on `learning-app-test` before the integration tests can pass.
- The real SMTP password in `.env` was shown in a chat. Rotate the Gmail app password.
- `src/modules/student/` exists but is empty.

## Next steps

- Build the student onboarding routes the frontend expects (`GET /student/onboarding`, the `PUT` step saves, `POST /student/onboarding/complete`, `GET /student/profile`). The shapes are in the frontend PR. Until then the app uses a dev only mock.
- Decide whether to remove the old password endpoints.
- Add admin routes later under `/admin`, following the new skill.

## Test plan

**Automated tests (40 unit tests passing, integration not run):**
- [x] `npm run typecheck`: passes
- [x] `npm test`: 6 files, 40 tests passing, including `otp.service.spec.ts`, `google-identity.service.spec.ts`, `auth-request.schemas.spec.ts` and `env.schema.spec.ts`
- [ ] `test/integration/email-otp.int-spec.ts` and `google-login.int-spec.ts` - not run. They need `0004` applied to `learning-app-test`
- [ ] `npm run lint` - not run for this doc

**Manual checks (local):**
- [x] Registered a user through the app flow earlier, which reached the database. After that I cleared the user rows on request
- [ ] Real email delivery through Gmail - not run
- [ ] Full signup, code login and dashboard on a device - not run
- [ ] `migrate:down` then `migrate:up` for `0004` - not run
