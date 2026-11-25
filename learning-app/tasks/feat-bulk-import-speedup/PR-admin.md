# 2026-11-20 - bulk-import-speedup (admin)

## Summary

The import page sent rows to the server one request at a time, 25 rows per request, and saved passages one after another. Long rows could also push a request over the server's 100 kb body limit. This PR sends bigger chunks sized by bytes as well as rows, runs two chunks at the same time, saves passages a few at a time, and adds an optional one-step "Fast direct import". The side menu also got a small layout change.

Passages are no longer saved by the browser. Each request carries the passage texts and the server saves them with the questions, so there are no extra passage calls and a rejected row leaves no passage behind.

Checked live: the admin unit tests pass (55 of 55) and typecheck passes. I did not open the page in a browser or time a real import. The backend side of the passage change is covered by integration tests.

## What changed

| File | Change |
|---|---|
| `src/features/content/lib/bulk-file.ts` | `CHUNK_SIZE` is 50 to match the backend. Added `CHUNK_MAX_BYTES` (70,000), `rowBytes()`, `chunkRows()` and `passagesOf()`. `toApiRow` now sends `passageKey` instead of `passageId`. The old `chunk()` is kept. |
| `src/features/content/lib/bulk-file.test.ts` | Tests for `chunkRows` (row cap, byte budget, oversized row, a shared passage counted once) and `passagesOf`. |
| `src/features/content/api/content.api.ts` | `runBulk` takes a `passages` map. |
| `src/features/content/pages/import-page.tsx` | `sendChunks` uses `chunkRows`, runs 2 chunks at once and sends the passage texts of each chunk. The passage saving step and its calls are gone. New "Fast direct import" checkbox and an animated dots progress indicator. The "Publish questions" checkbox now sits next to it. |
| `src/layouts/dashboard-layout.tsx` | The top bar is removed. The name, initial and a log out button now sit at the bottom of the side menu. Not part of the import work. |
| `src/components/common/app-sidebar.tsx` | Menu links get a pointer cursor. Not part of the import work. |

## Why

The backend now accepts 50 rows per request and handles each request faster. The admin needed to send bigger chunks and stop waiting on one request at a time, without going over the 100 kb body limit.

## How

**Chunks by rows and bytes**: `chunkRows` starts a new chunk when it reaches 50 rows or 70,000 bytes of row text. A single row that is bigger than the budget goes alone in its own chunk.

**Two at a time**: a small worker pool takes the next chunk when one finishes. Two keeps the server's connection pool free. Results are collected by row number, so order does not matter.

**Passages**: rows send a `passageKey` and each request carries the text of the passages its rows use, once per passage. The byte budget counts a passage once per request. The server reuses or saves the passage.

**Fast direct import**: when ticked, the file is imported in one pass with no separate check step and no "3. Import" card. Rows with errors found in the browser are shown as invalid and are not sent.

**Dry run now checks passage rows**: the check step used to send no passage for these rows. It now sends the passage, so the server checks them like any other row.

## Decisions made

**Direct import is a choice, not the default.** The two step flow lets the admin see problems before anything is saved. The one step flow is faster but saves passages before the rows are checked.

**Concurrency stays low (2 chunks).** More would compete for the 10 database connections on the backend.

## Notes

- This needs the backend change in `PR-backend.md` (the `passages` map and `passageKey`). Deploy the backend first.
- A passage with a key but no text in the whole file is sent as a row error by the browser check, as before.
- The side menu change and the cursor change are unrelated to the import and can be reviewed on their own.
- `npm run lint` shows only warnings in files this PR does not touch (`components/ui/button.tsx`, `tabs.tsx`, `badge.tsx`).

## Test plan

**Automated tests (55 passing):**
- [x] `src/features/content/lib/bulk-file.test.ts`: counts a shared passage once per request
- [x] `src/features/content/lib/bulk-file.test.ts`: lists each passage text once
- [x] `src/features/content/lib/bulk-file.test.ts`: chunks stop at the row cap
- [x] `src/features/content/lib/bulk-file.test.ts`: chunks stop at the byte budget
- [x] `src/features/content/lib/bulk-file.test.ts`: an oversized row is sent alone
- [x] `npx tsc --noEmit`: passes
- [x] `npm run lint`: no errors, warnings only in untouched files

**Manual checks (local):**
- [ ] Importing a large file in both modes and timing it - not run in this session
- [ ] The new side menu layout in the browser - not run in this session
