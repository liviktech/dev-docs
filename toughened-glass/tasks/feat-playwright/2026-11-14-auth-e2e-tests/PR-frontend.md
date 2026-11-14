# 2026-11-14 - auth-e2e-tests (frontend)

## Summary

The frontend had no browser tests. This PR sets up Playwright and adds an end to end suite for authentication: login, forgot password, redirects, session handling and mobile access. It also fixes a small bug where `process.env` was used in browser code. Playwright starts the backend and the frontend dev server itself unless they are already running.

I did not run the suite or the build for this write up, so nothing below is marked as checked live.

## What changed

| File | Change |
|---|---|
| `playwright.config.ts` (new) | Playwright setup: Chrome by default, loads `.env.test`, starts the backend (with `RATE_LIMIT_MAX=100000`) and the frontend dev server, records trace, screenshot and video for every test, 4 workers locally and 1 on CI. |
| `e2e/global-setup.ts` (new) | Before any test, checks the backend health endpoint and that the seeded `admin`, staff and super admin accounts can log in, and stops with a clear message if not. |
| `e2e/authentication/login.spec.ts` (new) | Login tests. |
| `e2e/authentication/forgot-password.spec.ts` (new) | Forgot password flow tests. |
| `e2e/authentication/redirects.spec.ts` (new) | Redirect tests for logged in and logged out users. |
| `e2e/authentication/session.spec.ts` (new) | Session and logout tests. |
| `e2e/mobile/mobile-access.spec.ts` (new) | Mobile access tests. |
| `e2e/fixtures/auth-helpers.ts`, `e2e/fixtures/mobile-helpers.ts`, `e2e/fixtures/test-users.ts` (new) | Shared helpers and the test account list. |
| `e2e/TEST-SCENARIOS.md` (new) | List of the scenarios the suite covers. |
| `.env.test.example` (new) | Template for the test accounts and URLs. |
| `src/main.tsx` | `process.env.VITE_ENV` changed to `import.meta.env.VITE_ENV` for the dev only `Agentation` widget. `process.env` does not exist in the browser build. |
| `package.json` | Added `@playwright/test` and the scripts `test:e2e`, `test:e2e:ui`, `test:e2e:headed`. The three `@dnd-kit` entries only moved to alphabetical order, no version change. |
| `package-lock.json` | Lock file update for the new dev dependency. |
| `tsconfig.node.json` | Adds the `DOM` lib and includes `playwright.config.ts` and `e2e/**/*.ts` so they are type checked. |
| `.gitignore` | Keeps `.env.test.example` tracked and ignores `playwright-report` and `test-results`. |

## Why

Login is the entry to the whole app and had no automated cover. Browser tests catch problems the API tests cannot, like redirects, cookies and the mobile view. The `main.tsx` fix was found while getting the config to type check.

## How

**Environment**: copy `.env.test.example` to `.env.test` to set the base URL and the three test accounts (admin, staff, super admin). All have defaults, so the file is optional if the seeded accounts are unchanged.

**Servers**: `playwright.config.ts` waits on `/api/v1/health` for the backend, which also runs `SELECT 1`, so it only goes green when the database is reachable. It reuses servers that are already running (not on CI). PostgreSQL itself is not started, and `npm run migrate` and `npm run seed` must already have been run in the backend repo.

**Rate limit**: the backend limits each IP to 300 requests per 15 minutes by default, so the config starts it with `RATE_LIMIT_MAX=100000`. If the backend is already running, set the same value in its `.env` and restart, or the limit can trip in the middle of the suite.

**Browser**: uses the installed Google Chrome. Set `PLAYWRIGHT_CHANNEL=` (empty) to use Playwright's bundled Chromium instead (`npx playwright install chromium`).

## Decisions made

**Trace, screenshot and video always on.** This makes each run heavy, but it gives full evidence for every test, passed or failed. If it is too slow, change it to `on-first-retry` or `retain-on-failure`.

**Global setup fails fast.** Without it, a missing seed user would make negative tests pass for the wrong reason (for example "invalid credentials" instead of "not a Super Admin"). `E2E_ALLOW_MISSING_STAFF` skips only the staff account check for partial runs.

## Notes

- The test file sizes are roughly: login 20, forgot password 17, redirects 8, session 13, mobile 22 test cases.
- The backend `037_e2e_customercare_user.sql` seed creates the `customercare` user used here, see `PR-backend.md`.

## Test plan

**Automated tests (not run for this write up):**
- [ ] `e2e/authentication/login.spec.ts`: login cases.
- [ ] `e2e/authentication/forgot-password.spec.ts`: forgot password cases.
- [ ] `e2e/authentication/redirects.spec.ts`: redirect cases.
- [ ] `e2e/authentication/session.spec.ts`: session and logout cases.
- [ ] `e2e/mobile/mobile-access.spec.ts`: mobile cases.
- [ ] `npm run build`: build and type check not run yet.
- [ ] `npm run lint`: not run yet.

**Manual checks (local):**
- [ ] `npm run test:e2e` against a migrated and seeded backend - not run yet.
- [ ] Open the app with `VITE_ENV=development` and confirm the `Agentation` widget still shows after the `import.meta.env` change - not run yet.
