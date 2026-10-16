# 2026-10-08 - photo-rotate-four-sides (backend)

> **Linked PR (frontend):** https://github.com/liviktech/rubine-toughened-glass-frontend/pull/102

## Summary

The AI used to read one width and one height per piece, and it was told to decide for itself which way up the photo was. Opposite sides that differ (2762 along the top, 2742 along the bottom) were squashed into one number, and a wrongly guessed orientation could swap width and height. This PR makes the AI treat the photo exactly as it receives it (staff now turn it upright before upload, see the frontend PR) and read all four sides of every piece: top width, bottom width, left length and right length.

Checked live on the local app against the real Gemini model: a fresh read of a tall sheet returned 8 rows with all four sides, and a rotated and re-uploaded sheet came back with different top and bottom widths (2762 and 2742) and its own left and right lengths. Readings saved before this change still open correctly.

**Debug and cookie edits that were already uncommitted in this repo are not included on purpose** - see "Notes" below. This was a choice, not a miss.

## What changed

| File | Change |
|---|---|
| `src/shared/vision-ai/result.schema.ts` | `width` and `height` replaced by `topWidth`, `bottomWidth`, `leftLength`, `rightLength` in the Zod row schema, the list of fields the AI may flag as unsure, and the JSON schema sent to Gemini, OpenAI and Anthropic. All four are required and above 0. |
| `src/shared/vision-ai/instruction.ts` | The "photo may be rotated" rule is replaced by "the photo is already upright, do not rotate or second guess it". New rules for the four sides. The "give the larger of opposite sides" rule is replaced by "each side keeps its own figure". The analysis step now names which side each figure belongs to. Examples, the "rows merge only when everything matches" rule and the reply shape use the four sides. |
| `src/modules/ai-extraction/row-cleanup.ts` | Converts all four sides to mm. Adds `originalSides` (the four inch values as written) and keeps `originalSize` for the Remark text. New `normalizeStoredRow()` turns an old one width, one height row into the four side shape. |
| `src/modules/ai-extraction/ai-extraction.types.ts` | `CleanRow` has `topWidthMm`, `bottomWidthMm`, `leftLengthMm`, `rightLengthMm` and an optional `originalSides`. New `OriginalSides` type. |
| `src/modules/ai-extraction/ai-extraction.schema.ts` | The confirm route accepts the four sides (each above 0) and the optional `originalSides`. |
| `src/modules/ai-extraction/ai-extraction.service.ts` | Readings and confirmed rows go through `normalizeStoredRow()` when read from the database. The "confirm with a target" path writes Actual Width as the larger of top and bottom, and Actual Height as the larger of left and right. |
| `src/docs/swagger/ai-extraction.swagger.ts` | The row example shows the four sides. |
| `src/app.ts` | no real change from this PR (a debug `console.log` that was already uncommitted, not touched) |
| `src/modules/auth/auth.controller.ts`, `src/modules/auth/auth.service.ts` | no real change from this PR (already uncommitted, not touched) |

## Why

The Add Product form already has Top Width, Bottom Width, Left Length and Right Length, and works out the overall size as the larger of each pair. The AI reading only filled one width and one height and copied them to all four, so a sheet with an out of square opening lost information. The orientation rule was also a weak point: a model that guessed "sideways" would swap width and height for every row. Staff can turn the photo themselves in seconds, so the model is now told to trust it.

## How

**Four sides, as the photo is held**: top and bottom are the edges that run across the photo, left and right the edges that run up and down. Which side is which comes from where the edge sits, never from which number is bigger. One figure written once for a pair goes to both sides of the pair, which is what a plain rectangle needs. Different figures on a pair each keep their own. A circle gets its diameter on all four. A side with no figure on a shape that is not a plain rectangle is worked out, flagged in `unsureFields` and explained in `note`.

**Photo orientation is given**: the instruction says the photo is already the right way up and must not be rotated or questioned. Nothing is stored about the turn, because the turn is baked into the image bytes by the browser (see the frontend PR).

**Older readings keep working**: a stored row with one `widthMm` and one `heightMm` is converted when it is read. Top and bottom get the old width, left and right get the old height. No migration, nothing in the database is rewritten, and the reading JSON files next to the images are not touched.

**Inch sheets**: `originalSides` holds the four inch values exactly as read, so the review panel can show each one under its own field. Converting back from the rounded mm value would show 36.26 instead of 36.25.

**Kept the same on purpose**: the database tables and columns, the stored reading shape (rows are JSON), the routes, rights, rate limit, the reuse by image fingerprint, and the `originalSize` text used in the Remark.

## Decisions made

**Required fields, not optional.** All four sides are required in the AI reply, and the model is told to repeat the figure for a pair. Optional sides would let a model leave gaps that the review table then has to guess.

**Old readings are converted when read, not migrated.** A migration would rewrite JSON in two tables for readings that may never be opened again. Converting on read costs almost nothing and is easy to remove later.

**The larger of each pair is the overall size.** This matches the Size Details tab, so a draft saved through Add Product and rows written through the confirm target path agree.

**The "give the larger of opposite sides" rule is gone.** It was an assumption in the earlier prompt refinement PR (decision 4) that still needed the client to confirm. With four sides there is nothing to pick, so that question no longer has to be asked.

**`originalSize` is kept.** The Remark text built by the Add Product form and by the server path still uses it. `originalSides` is added next to it, not instead of it.

## Notes

- Local database: the developer database was 14 migrations behind the code, so most screens failed with missing column errors. It was **not** migrated. The live test used a copy (`rubine_scratch`) that was migrated, with the backend started with `DATABASE_URL` pointing at it. The copy can be dropped.
- Stored readings are keyed by image bytes, so photos already read keep their old result (now shown in the four side shape) until **Read again** is used.
- The reading JSON written next to an image still holds the rows exactly as the database does. An older file keeps the old shape.
- Read cost seen live: one fresh read used 6,717 input and 6,245 output tokens and took about 18.5 seconds on `gemini-3.5-flash-lite` with `GEMINI_THINKING_LEVEL=high`.
- Already uncommitted in this repo before this task and **left out of this PR**: a debug `console.log` in `src/app.ts`, `console.log` lines in `src/modules/auth/auth.service.ts` that print the access and refresh tokens, and a cookie change in `src/modules/auth/auth.controller.ts` (`secure: true`, `sameSite: "none"`). The token logging in particular should not be committed.
- The product docs were updated in the same task: `dev-docs/toughened-glass/features/ai-extraction/AI_EXTRACTION_DEVELOPER_GUIDE.md` and `AI_EXTRACTION_INFO.md`.

## Bugs found while testing (already there before, not fixed here)

**1. Migrations cannot be replayed on an empty database.** `npm run migrate` on a new empty database stops at `030_inventory_schema.sql` because it needs `public.work_order`, which does not exist at that point. The seed `012_inventory_seed.sql` then fails too. It only works on a database that already has the older tables. This affects anyone setting up a fresh local database.

## Next steps

- Run **Read again** on a few real sheets with each vendor the client might pick (OpenAI, Anthropic, Mistral, Qwen) to check they all return the four sides. Only Gemini was run.
- Fix or document the empty database migration problem above.
- Remove the token `console.log` lines from `auth.service.ts` before anyone commits that file.

## Test plan

**Automated tests (no automated test suite exists in this repo):**
- [x] `npx tsc --noEmit`: clean, no errors.

**Manual checks (local, real Gemini model, backend on the migrated copy of the database):**
- [x] Fresh read ("Read again") of the tall sample `10.jpg`: 8 rows, each with all four sides, values matched the sheet (for example 1895 x 1971 and 772 x 1967). Stored as `source = 'ai'`, status `ready`.
- [x] Rotated upload: `12.jpeg` (720x1280) turned right and uploaded. The file on disk is 1280x720. The read gave 2 rows with top 2762 and bottom 2742, left 2372 and right 2372, and a second row with top 2754 and bottom 2734.
- [x] Older reading: opening `1.jpg` and `10.jpg` (readings saved before the change) showed top equal to bottom and left equal to right.
- [x] Add Product form: a draft from a read opened with Top Width 1895, Bottom Width 1895, Left Length 1971, Right Length 1971, and the area calculated as 3.735.
- [ ] Confirm with a target (`orderBookingId` and `itemId`), which uses the larger of each pair for Actual Width and Height - not run, the UI never uses this path today. Covered by the type check only.
- [ ] Read on OpenAI, Anthropic, Mistral and Qwen with the new fields - not run, no keys for them on this machine. The schema is shared, so they receive the same fields.
- [ ] Read with `GEMINI_THINKING_LEVEL` empty, and a read while the product list query fails - not run (carried over from the earlier prompt refinement PR).
