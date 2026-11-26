# 2026-11-25 - jest-test-suite

## Summary

The mobile app had no automated tests, so a broken screen, hook or API call was only found by running the app by hand. This PR adds a Jest test setup (jest-expo and React Native Testing Library) with 88 tests across 17 files, covering the API client, auth service, domain APIs, hooks, shared components and the main screens. The pre-push hook now runs the tests as well as lint, so a failing test blocks a push. All 88 tests pass, lint and the type check are clean, and coverage is above the thresholds.

**Backend tests are not included on purpose** - see "Decisions made" below. This was a choice, not a miss.

## What changed

### learning-app (mobile)
| File | Change |
|---|---|
| `jest.config.js` (new) | jest-expo preset, test file pattern, `@/` path mapping, asset and style mocks, and coverage thresholds (80 statements, 75 branches, 80 functions, 80 lines). |
| `package.json` | Adds Jest, jest-expo, React Native Testing Library and related dev dependencies. Adds `test`, `test:watch`, `test:coverage` and `test:ci` scripts. |
| `yarn.lock` | Lock file update for the new dev dependencies. |
| `tsconfig.json` | Adds `jest` to `types` so test globals type check. |
| `.githooks/pre-push` | Now runs `npm run lint` and then `npm test`. A failure in either blocks the push. |
| `.gitignore` | Ignores test output. |
| `.claude/skills/pre-push-checks/SKILL.md` (new) | Short guide for running lint and tests before a push. |
| `.claude/skills/write-tests-for-new-features/SKILL.md` (new) | Guide that asks for tests with every new feature. |
| `__tests__/setup/jest.setup.ts` (new) | Mocks for native modules (Firebase, Google sign in, Rive, Skia, router, secure store and others) and test env variables. |
| `__tests__/setup/fixtures.ts` (new) | Shared fake data for users, questions, exams, plans and stats. |
| `__tests__/setup/test-utils.tsx` (new) | Render helper that wraps components in the query client and other providers. |
| `__tests__/setup/mocks/asset-mock.js`, `style-mock.js` (new) | Stubs for `.riv` and `.css` imports. |
| `__tests__/services/api-client.test.ts` (new) | 8 tests for the API client. |
| `__tests__/services/auth-service.test.ts` (new) | 6 tests for the auth service. |
| `__tests__/services/domain-apis.test.ts` (new) | 6 tests for the domain API modules. |
| `__tests__/utils/formatters.test.ts` (new) | 5 tests for the formatting helpers. |
| `__tests__/hooks/hooks.test.tsx` (new) | 8 tests for the data hooks. |
| `__tests__/components/*.test.tsx` (3 files, new) | 16 tests for UI, auth and onboarding, and feature components. |
| `__tests__/screens/**/*.test.tsx` (10 files, new) | 39 tests for auth, onboarding, dashboard and plan, practice, mock exams, results and stats, bookmarks, profile and parent screens. |
| `src/app/(tabs)/tests.tsx` | One unused line removed so lint stays clean. No behavior change. |
| `src/components/ui/book-loader.tsx` (new), `src/components/ui/loading-state.tsx` | Not part of the tests. This is the separate BookLoader loading animation already on this branch. |

## Why

Every change so far was checked by hand on a device or emulator. With rate limits, audit headers, parent login and onboarding all landing, it is easy to break an older screen without noticing. A test suite gives a fast check before each push and a safety net for the next features.

## How

**Setup**: `jest-expo` runs the tests with the same Babel and module rules as the app. `jest.setup.ts` replaces native modules with simple mocks, so tests run in plain Node with no device.

**What gets mocked**: only the outside edges (network calls, native modules, router, storage). The code under test, such as the API client, hooks and screens, runs for real. Screen tests mock the API layer and check what the user sees and which calls are made.

**Shared helpers**: `fixtures.ts` holds fake data and `test-utils.tsx` holds one render helper, so each test file stays short.

**Guardrails**: coverage thresholds fail the run if coverage drops. The pre-push hook runs lint and then tests, so broken code does not reach the remote.

## Decisions made

**Backend tests left out of this PR.** The backend already has its own test setup (Vitest) and this PR is only about the mobile app. Mixing them would make the review harder.

**Mocked the API layer in screen tests, not a fake server.** A fake server would be closer to real life but is slower and needs more setup. API client behavior is covered separately in `api-client.test.ts`.

**No snapshot tests.** They break on small style changes and say little. Tests check text, buttons and calls instead.

**Coverage thresholds set at 80/75/80/80.** Current coverage is above them, so they act as a floor without blocking this PR.

## Bugs found while testing (already there before, not fixed here)

**1. Tests leak a timer.** Jest prints "A worker process has failed to exit gracefully" after the run. Some code starts a timer or listener that is never cleared in tests. It does not fail any test. Running with `--detectOpenHandles` should point at the source. Affects only test run time and noise.

## Notes

- Run everything with `npm test`. Use `npm run test:coverage` for the coverage table.
- The hook is installed by `yarn install` through the `prepare` script, so teammates get it after pulling and installing.
- Coverage on the last run: 89.28 statements, 77.13 branches, 85.53 functions, 90.65 lines.
- `learning-app/yarn.lock` shows a large diff only because of the new dev dependencies.

## Next steps

- Find the leaking timer with `--detectOpenHandles`.
- Add a CI step that runs `npm run test:ci`, so tests also run for people who skip the hook.
- Raise the branch coverage threshold once more error paths are covered.
- Add tests for the remaining screens (admin-style flows and settings) when they change.

## Test plan

**Automated tests (88 passing, 17 suites):**
- [x] `__tests__/services/api-client.test.ts`: 8 tests, passing
- [x] `__tests__/services/auth-service.test.ts`: 6 tests, passing
- [x] `__tests__/services/domain-apis.test.ts`: 6 tests, passing
- [x] `__tests__/utils/formatters.test.ts`: 5 tests, passing
- [x] `__tests__/hooks/hooks.test.tsx`: 8 tests, passing
- [x] `__tests__/components/*.test.tsx`: 16 tests across 3 files, passing
- [x] `__tests__/screens/**/*.test.tsx`: 39 tests across 10 files, passing
- [x] `npx tsc --noEmit`: no errors
- [x] `npm run lint`: no errors
- [x] `npm run test:coverage`: thresholds met (89.28 / 77.13 / 85.53 / 90.65)

**Manual checks (local):**
- [x] Ran `npm test` from a clean state: 17 suites, 88 tests, all passing in about 17 seconds
- [ ] Push with a deliberately failing test to confirm the hook blocks it - not run, the hook script is a plain lint then test sequence with exit 1 on failure
- [ ] Run on CI - no CI test step exists yet, see "Next steps"
