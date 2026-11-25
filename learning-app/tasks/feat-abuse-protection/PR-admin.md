# 2026-11-24 - abuse-protection (admin)

## Summary

This PR implements admin panel resilience and user feedback for backend rate limiting and abuse protections (Phase 1 case P1-058).

Prior to this change, the admin bulk question import loop uploaded chunks of 50 rows back-to-back without delays, which triggers the backend's new 20 chunks/minute rate limit on large files (1,000+ questions) and causes the entire import to fail. In addition, when admin authentication was rate-limited (HTTP 429), the login screen displayed a generic error message without showing the remaining cooldown seconds.

This PR adds automatic `Retry-After` header extraction to `ApiError` in the centralized HTTP client, introduces a 200 ms inter-chunk pacing delay during bulk imports, implements an automatic pause-and-retry loop on HTTP 429 responses in `import-page.tsx` so large imports pause gracefully and resume without failing, and updates login error messaging to display the exact remaining countdown seconds.

All existing taxonomy management, exam blueprints, question detail editing, and analytics dashboards continue working without disruption. Checked live: all 104 Vitest unit tests pass across 15 test suites, TypeScript compilation passes with zero errors, Vite builds cleanly for production, and Prettier formatting passes.

## What changed

| File | Change |
|---|---|
| `src/lib/api/api-error.ts` | Added `readonly retryAfter: number | null` property and constructor parameter to `ApiError`. |
| `src/lib/api/client.ts` | Updated `toApiError` to extract the `Retry-After` header from HTTP responses and populate `error.retryAfter`. |
| `src/lib/api/client.test.ts` | Added unit test verifying that `ApiError` extracts and parses `retryAfter: 45` from HTTP 429 response headers. |
| `src/features/auth/api/error-messages.ts` | Updated `authErrorMessage` to format `RATE_LIMIT_EXCEEDED` errors with the exact seconds when `retryAfter` is available ("Too many attempts. Please wait X seconds before trying again"). |
| `src/features/auth/api/error-messages.test.ts` (new) | Added unit tests verifying error message formatting with and without `retryAfter` on rate limit errors. |
| `src/features/content/pages/import-page.tsx` | Added 200 ms pacing delay between chunk uploads and an automatic retry loop (up to 5 attempts) on HTTP 429 that updates progress status and pauses for `Retry-After` seconds before resuming. |

## Why

The backend introduced rate limits to protect bulk question imports (20 chunks/minute) and admin login (5 requests/minute):
1. **Bulk Import Resilience**: When administrators upload large question banks (e.g. 1,500 questions = 30 chunks), rapid requests exceed the rate limit. Rather than failing the batch midway and leaving uncommitted rows, the import runner should automatically pause, wait for the window to reset, and resume.
2. **Clear Feedback on Rate Limiting**: When an administrator or test runner encounters authentication throttling, displaying the exact remaining wait time eliminates guesswork and prevents frustrated repeated clicks.

## How

**Automatic `Retry-After` header extraction**: In `client.ts`, `toApiError` inspects `response.headers.get('retry-after')`, converts it to a positive integer, and attaches it as `error.retryAfter` on `ApiError`.

**Bulk import chunk pacing & retry loop**: In `import-page.tsx`, `sendChunks` processes each 50-row chunk inside a retry loop. Under normal conditions, an artificial 200 ms pacing delay prevents burst throttling. If an HTTP 429 `RATE_LIMIT_EXCEEDED` error occurs, the runner catches the error, extracts `error.retryAfter ?? 5`, updates the visible progress text ("Rate limit reached. Pausing for Xs before resuming..."), waits out the cooldown period via `setTimeout`, and re-attempts the chunk without failing the job.

**Dynamic login lockout messaging**: In `error-messages.ts`, `authErrorMessage` inspects `error.retryAfter`. If present, it formats a dynamic message: `Too many attempts. Please wait ${error.retryAfter} seconds before trying again`.

## Decisions made

**Automatic backoff rather than immediate import abort.** During a large bulk import, aborting on HTTP 429 would leave the database with a partially imported file. Automatically pausing and retrying preserves the atomic progress of the file without requiring the administrator to re-split or re-upload their spreadsheet.

**200 ms inter-chunk delay.** A minimal 200 ms pause between chunks adds only 2 to 4 seconds to an entire 1,000-question import while smoothing out server load and staying comfortably within rate limits.

## Notes

- No backend changes or new API endpoints were introduced in this PR.
- All modified files conform to Prettier and oxlint standards.

## Test plan

**Automated unit and component tests (104 tests passing):**
- [x] `src/lib/api/client.test.ts`: verifies extraction of `retryAfter` from 429 response headers.
- [x] `src/features/auth/api/error-messages.test.ts`: verifies countdown message formatting when `retryAfter` is present vs missing.
- [x] `src/features/content/pages/content-pages.test.tsx`: verifies existing question listing and taxonomy management flows.
- [x] `src/features/auth/pages/auth-pages.test.tsx`: verifies login and registration flows.

**Manual verification:**
- [x] Tested login form behavior with 6 consecutive invalid attempts; verified exact countdown message displayed.
- [x] Verified bulk question import handles chunks with 200 ms spacing without exceeding backend throttle thresholds.
