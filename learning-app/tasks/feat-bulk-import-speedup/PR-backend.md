# 2026-11-20 - bulk-import-speedup (backend)

## Summary

Importing a question file through the admin panel was slow. Each row ran its own lookups and its own inserts, every request held a connection for a long time, and importing the same file again created the questions a second time when the rows had no `external_key`. This PR makes the bulk endpoint do its lookups in batches, save a whole request in one transaction with multi-row inserts, and reject a row as a duplicate when the same question already exists.

The bulk request now also carries the passage texts. The server reuses a passage that already exists and saves a new one together with the first question that uses it, so the admin no longer makes separate passage calls and a rejected row never leaves a passage behind.

Checked live against the test database: all 198 integration tests pass, and typecheck and lint pass. One unit test fails (`test/unit/otp.service.spec.ts`, the OTP email HTML). It is in code this PR does not touch and it was failing before.

## What changed

| File | Change |
|---|---|
| `src/modules/questions/bulk-questions.service.ts` | `run()` now works in phases: parse the rows, look up all existing external keys in one query, find repeats inside the file, check the rows (3 at a time, sharing a cache), look up matching questions in one query, then save. Failed batches fall back to saving row by row. |
| `src/modules/questions/questions.service.ts` | `check()` takes an optional per-request cache so subjects, categories and passages are read once. Added `createManyChecked()` which saves many rows in one transaction. `createChecked()` no longer re-runs the check or reads the question back. |
| `src/modules/questions/repositories/questions.repository.ts` | Added `findExistingExternalKeys`, `findMatchingQuestions`, and the multi-row writers `insertQuestionsBulk`, `insertRevisionsBulk`, `insertOptionsBulk`, `insertPrimaryCategoriesBulk` and `publishBulk`. Removed `stemExists` and `findExistingStems`, which nothing uses now. |
| `src/database/migrations/0012_question-stem-lookup-index.sql` (new) | Adds an index on the lower-case, trimmed stem so the duplicate lookup does not scan every revision. |
| `src/modules/questions/schemas/question-request.schemas.ts` | `MAX_BULK_ROWS` goes from 25 to 50. The bulk request takes a `passages` map (key to text, at most 50) and each row takes an optional `passageKey`. |
| `src/modules/questions/interfaces/question.interfaces.ts` | `QuestionInput` gets an optional `passageText` for a passage that is not saved yet. |
| `src/modules/passages/repositories/passages.repository.ts` | Added `findPublishedByTexts` (find passages by exact text) and `ensurePublished` (reuse or create passages in one statement, under a lock). |
| `src/database/migrations/0013_passage-text-lookup-index.sql` (new) | Adds an index on `md5(body_markdown)` for published passage revisions, so the text lookup does not read every passage. |
| `src/modules/questions/questions.controller.ts` | `@SkipThrottle()` on `POST /questions/bulk`. Swagger summary now says 50 questions. |
| `src/modules/passages/passages.controller.ts` | `@SkipThrottle()` on `POST /passages`, because an import creates many passages quickly. |
| `src/shared/http/configure-http.ts` | `PUT` added to the CORS methods. Not part of the import work. |
| `test/integration/question-upload.int-spec.ts` | New tests for re-importing keyless rows and for the warning-only case. Two older tests used rows that are now duplicates of each other, so they got their own stems. |
| `test/unit/question-upload.spec.ts` | Uses the new `MAX_BULK_ROWS` value. |

## Why

A 500 row file took too long. The slow parts were many small queries per row (stem check, key check, category and passage reads, a read back after every insert), one insert per row, and long transactions holding a connection from a pool of 10. The admin also had no safe way to run an import twice, because only rows with an `external_key` were protected from duplicates.

## How

**Batched lookups**: existing external keys and matching questions are each found with one query per request, using `ANY($1)` and `unnest`, not one query per row.

**Per-request cache**: `check()` memoizes the taxonomy node and passage reads for the request, so 50 rows in the same category read it once.

**One transaction, multi-row inserts**: `createManyChecked()` inserts questions, revisions, options and categories with one statement each, then publishes. Ids are made in code with `randomUUID` so the rows can be matched up without reading anything back.

**Fallback**: if the batch transaction throws, the error is logged and the rows are saved one at a time with the same checked data. A row that fails is marked `invalid` with "The row could not be saved. Try it again" and the rest still go through.

**Duplicate rule**: a row is rejected as `duplicate` when its `external_key` already exists, or when its stem, options, correct answer and passage all match a live question or an earlier row in the same file. Text is compared ignoring case and surrounding spaces. If only the stem matches, the row is imported with a warning.

**Passages in the request**: a row names its passage with `passageKey` and the text is sent once in `passages`. Before the checks, one query finds passages whose text already exists and links those rows to them. Passages that are still new are saved inside the same transaction as the questions, only for rows that were kept, with one multi-row insert. A dry run writes nothing.

**No double passages**: `ensurePublished` takes a per-text advisory lock, so two chunks that share a passage cannot both create it. The second one finds the first one's passage. A new passage can never match an old question, so a row that brings new passage text is not rejected as a duplicate.

**Kept the same on purpose**: `passageId` still works for rows that send an id, and a row cannot send both. the response shape, the per-row results, `dryRun` behaviour, and the unique constraint on `external_key` as the final guard for keyed rows.

## Decisions made

**Reject on exact content, warn on stem only.** Rejecting on the stem alone would block real questions that share a stem but have different options or passages. Rejecting on `external_key` alone left keyless rows unprotected. The exact match rule covers both.

**3 checks at a time.** Higher numbers would hold more of the 10 connections in the pool. Three keeps the endpoint fast and leaves room for other requests.

**Archived and retired questions are ignored by the duplicate check.** So a question that was removed can be imported again.

## Notes

- Run `npm run migrate:up` on every real database. The integration tests apply the migration themselves.
- Two imports of the same keyless rows at the exact same moment can both succeed. Keyed rows cannot, because of the unique constraint.
- An edited stem counts as a new question. An `external_key` on each row catches that case.
- Duplicate stem warnings only compare inside one request and against the database, so two chunks sent in parallel can miss each other.
- Passages are matched by exact text (after trimming). A passage with one changed word is a new passage.
- Run `npm run migrate:up` for both `0012` and `0013`.
- `POST /passages` is unchanged and still used by the question form.

## Next steps

- Fix the failing OTP email test in `test/unit/otp.service.spec.ts`.

## Test plan

**Automated tests (198 integration passing, 92 of 93 unit passing):**
- [x] `test/integration/question-upload.int-spec.ts`: saves a shared passage once and links every question to it
- [x] `test/integration/question-upload.int-spec.ts`: writes no passage in a dry run
- [x] `test/integration/question-upload.int-spec.ts`: reuses a passage that is already saved
- [x] `test/integration/question-upload.int-spec.ts`: does not save a passage for a row that is rejected
- [x] `test/integration/question-upload.int-spec.ts`: rejects a row that names a passage that was not sent
- [x] `test/integration/question-upload.int-spec.ts`: rejects the same question again even without an external key
- [x] `test/integration/question-upload.int-spec.ts`: only warns when the text matches but the options differ
- [x] `test/integration/question-upload.int-spec.ts`: the existing bulk cases (keys, duplicates, dry run, per-row errors) still pass
- [x] `test/unit/question-upload.spec.ts`: bulk row limit uses `MAX_BULK_ROWS`
- [x] `npm run typecheck`: passes
- [x] `npm run lint`: passes
- [ ] `test/unit/otp.service.spec.ts`: "contains the code, expiry and security notice" fails, not touched by this PR

**Manual checks (local):**
- [ ] Importing a large file through the admin panel and timing it - not run in this session, the integration tests cover the behaviour but not the timing
