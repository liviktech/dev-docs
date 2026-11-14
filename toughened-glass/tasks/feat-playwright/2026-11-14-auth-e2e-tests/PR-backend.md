# 2026-11-14 - auth-e2e-tests (backend)

## Summary

The backend had no automated tests for the login and session flow, and the global rate limit (300 requests per 15 minutes per IP) was hard coded, so a full browser test run would hit it half way. This PR adds API, middleware and schema tests for auth, makes the rate limit configurable with an env value, returns a proper 400 for broken JSON bodies, and adds a seed for the non-admin test user. The default behaviour in production does not change (limit stays 300).

I did not run the new tests or the build for this write up, so nothing below is marked as checked live.

**Fixing the auth findings is not included on purpose** - see "Bugs found while testing" below. This PR only adds tests and test support. It does not change login, token or company checks.

## What changed

| File | Change |
|---|---|
| `src/config/env.ts` | New `RATE_LIMIT_MAX` setting (whole number, default 300). |
| `src/middlewares/rate-limit.middleware.ts` | The global limiter now reads `env.RATE_LIMIT_MAX` instead of the fixed 300. |
| `.env.example` | Documents `RATE_LIMIT_MAX` (commented out, default 300). |
| `src/middlewares/error.middleware.ts` | Broken JSON bodies (`entity.parse.failed` from `express.json()`) now return 400 `VALIDATION_ERROR` with "The request body is not valid JSON." Before, they fell through to the 500 handler and were logged as server errors. |
| `package.json` | The `test` script now runs `tests/**/*.test.ts` instead of `src/**/*.test.ts`. |
| `tests/auth/auth-api.test.ts` (new) | Calls the real Express app and the real database: login, `/me`, refresh, logout, forgot password endpoints. |
| `tests/auth/auth-middleware.test.ts` (new) | Tests for `requireAuth` and token handling. |
| `tests/auth/auth-schema.test.ts` (new) | Tests for the Zod schemas in `auth.schema.ts`. |
| `src/database/seeds/037_e2e_customercare_user.sql` (new) | Adds the `customercare` user (role `CUSTOMER_CARE`) if missing. Safe to re-run, it does nothing when the user exists. |

## Why

The frontend Playwright auth suite needs a running backend with known users and a rate limit that will not trip. The backend also needed its own auth tests so the login rules are written down and protected. The work is mostly test support. The real change in behaviour is the 400 for bad JSON, which was a client mistake being reported as a server error.

## How

**Rate limit**: `RATE_LIMIT_MAX` is read through the existing env schema. Production and normal dev keep the default of 300. Playwright starts the backend with `RATE_LIMIT_MAX=100000`. If you already have the backend running, add that value to its `.env` and restart.

**Bad JSON handling**: the check sits in the error handler before the generic 500 path, so a client sending broken JSON gets a clear 400 and the log is not filled with fake server errors.

**API tests**: they use the real app and database, with the users `admin`, `customercare` and `superadmin` (override with `TEST_ADMIN_USER`, `TEST_STAFF_USER`, `TEST_SUPERADMIN_USER` and the matching `*_PASSWORD` values). Nothing is created or changed. The forgot password tests only use a number that no login can own (`0000000000`), bad input or fake tokens, so no OTP is created or sent.

**Seed**: older databases were seeded before `010_rbac_seed_data.sql` got its default users block, so they never received `customercare`. The new seed fills that gap and uses `ON CONFLICT (company_id, username) DO NOTHING`.

## Decisions made

**Moved the test glob to `tests/`.** The old script looked at `src/**/*.test.ts`. The new auth tests live in a top level `tests/` folder so test code stays out of the source tree. Any test files still under `src/` will no longer run with `npm test`, so move them if there are any.

**Seed file instead of editing 010.** Migrations and seeds are append only, so the missing user goes in a new seed file and the old one is left alone.

**Limit read from env, not a second limiter.** This keeps one global limiter and one setting. A separate stricter login limiter is a different change (see Next steps).

## Bugs found while testing (already there before, not fixed here)

These come from the security review in `info/docs/Rubine-glass/security-and-reliability-review.md`, checked against the code. None are changed by this PR.

**1. Username is not unique across companies.** Login looks up by username only and takes the first row, but the database only requires a unique username per company. Two companies with `admin` can log into the wrong one. Affects every multi company setup.

**2. Inactive companies can still log in.** `SELECT_APP_USER` never reads the company's `is_active`, and `login`, `getCurrentUser` and `refreshSession` only check the user. Existing sessions and refresh tokens (7 days) also keep working.

**3. No login specific rate limit.** Only the global limit applies to `POST /auth/login`, so password guessing is only slowed by 300 requests per 15 minutes per IP. `trust proxy` is also not set in `app.ts`, so behind a proxy all users may share one IP.

**4. Logout does not revoke tokens.** It only clears cookies. Access tokens last 1 hour by default and refresh tokens 7 days, and neither is checked against a blocklist.

## Notes

- The auth API tests need a migrated and seeded database. They are safe on a dev database.
- Never seed `037_e2e_customercare_user.sql` into production, the password is `password123`.
- `RATE_LIMIT_MAX` should stay at 300 (or the default) in production.

## Next steps

- Fix the four items above in a separate PR (company code on login, company active check in all three places, login limiter, token revocation).
- Add tests for each of those fixes once they land.

## Test plan

**Automated tests (not run for this write up):**
- [ ] `tests/auth/auth-api.test.ts`: login, `/me`, refresh, logout and forgot password against the real app and database.
- [ ] `tests/auth/auth-middleware.test.ts`: `requireAuth` and token cases.
- [ ] `tests/auth/auth-schema.test.ts`: Zod schema cases.
- [ ] `npm run build`: type check not run yet.

**Manual checks (local):**
- [ ] Send a request with a broken JSON body and confirm a 400 with code `VALIDATION_ERROR` - not run yet.
- [ ] Start the backend with `RATE_LIMIT_MAX=100000` and confirm the limit header changes - not run yet.
- [ ] Run `npm run seed` twice and confirm the `customercare` user is created once - not run yet.
