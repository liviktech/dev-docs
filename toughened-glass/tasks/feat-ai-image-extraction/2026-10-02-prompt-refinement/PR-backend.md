# Prompt refinement — backend

`feat/ai-image-extraction` → `dev` · repo `rubine-toughened-glass-backend` · commit `3647372`

> **Linked PR (frontend):** `<paste FE PR URL here>` — `prompt-refinement-PR-FE.md`
> No API contract changes between the two, so they can merge in either order. The frontend PR makes the longer remarks this one produces readable.

## Summary

Rewrites the instruction sent to the vision model so hand-drawn glass sheets are read the way a site engineer would read them, and makes the model reason before it answers.

On nine real, difficult sheets the previous prompt confused **cutouts with holes**, duplicated or merged pieces, copied struck-out lines into the order, and dropped the glass type from most rows. After this change all nine read as expected on repeat runs (table in *Test plan*).

Also in this commit: the master product list is pasted into the prompt, order bookings get their `order_code` again, and a safety net rejects an impossible glass thickness.

## What Changed

| Area | Change |
|---|---|
| `src/shared/vision-ai/instruction.ts` | Now `buildOrderSheetInstruction(vocabulary?)`, built per read. Rewritten around a **border test** for hole vs cutout, strict **row-identity** rules, struck-through writing, headings that cover many rows, notes written once, panes that add up, and worked examples. Reply must start with an `analysis`. |
| `src/modules/ai-extraction/product-vocabulary.ts` | **New.** Reads active `master.product` + `master.sub_product` names, holds them 10 min in memory, never throws (falls back to the last list, or none). |
| `src/shared/vision-ai/result.schema.ts` | Leading `analysis` string, required in the JSON schema so the model writes it before the rows. Only logged (debug), never stored or shown. |
| `src/shared/vision-ai/vision-ai.factory.ts` | `readOrderSheet(photo, vocabulary?)` builds the prompt per call; logs `analysis`. |
| `src/shared/vision-ai/providers/gemini.provider.ts`, `config/env.ts`, `.env.example` | New optional `GEMINI_THINKING_LEVEL` (`minimal`/`low`/`medium`/`high`), passed as `thinkingConfig`. |
| `src/modules/ai-extraction/row-cleanup.ts` | Thickness above 30 mm is dropped and flagged — it is a size in the wrong field. |
| `src/modules/ai-extraction/ai-extraction.service.ts` | Fetches the vocabulary and passes it to the read. |
| `src/modules/order-booking/order-booking.repository.ts` | **Unrelated fix:** `createOrderBooking` now sets `order_code` itself (see *Notes*). |
| `CLAUDE.md` | Documents the prompt builder, vocabulary and thinking setting. |

`ai-extraction-plans/` (`plan.md`, `requirements.md`) is updated alongside, outside this repo.

## Why

The numbers on the sheet *are* the order, and the reading is only useful if staff can trust it enough to review rather than redo it. The failures were not random: the model judged marks by **shape** (round = hole), when the physical question is whether the mark touches the border of the glass. Copying a heading onto only the first row also broke grouping on the Products table, because rows without a thickness fall into separate groups.

Small vision models also get these decisions wrong when they answer immediately and right when they have looked at each mark in words, which is what the `analysis` and the thinking level buy.

## How

- **Border test.** A mark that is only part of a circle, oval or box, with the glass border as its missing side, is a **cutout**; a complete closed mark inside the glass is a **hole**; an X inside is a hole, flagged. A mark on the line shared by two panes is a cutout in **each** pane. Dimension lines and arrows are never marks, and `Ø22mm` is a legend, not a hole.
- **Rows merge only when everything matches** — size, thickness, glass type, edge work, hole count, cutout count and position. A line across a drawing from edge to edge splits it into pieces; a line that stops at a dot is a leader line.
- **Struck-through writing is deleted**; a heading covers every line it spans and a repeated heading is the same product; every row carries the thickness, glass type and edge work that cover it.
- **Analysis first.** The model lists the glass descriptions, struck-out items, each piece with each mark and the reason it was judged, then totals checks; rows must follow it.
- **Products.** The active product names (with variants) are pasted in as vocabulary, with a brake: never force a match, never fill `productName` from the list.
- **Thinking.** `GEMINI_THINKING_LEVEL=high` adds a few thousand output tokens per photo (well under a cent) and 10–20 s per read.

## Decisions made

1. **Reasoning in the reply, not only in the prompt.** Wording alone fixed most cases; the hardest sheet (door leaf under a 12" transom) only came right with thinking on. Both are kept.
2. **`analysis` is logged, not stored.** No migration, no UI, and the stored reading keeps its current shape. If we want it for tuning later it is one column.
3. **A cutout across two panes counts once in each pane.** Each pane has its half. The earlier prompt said "once"; that was wrong for how these are drawn.
4. **Opposite sides with different numbers** (2762 vs 2742): give the larger, put both in the remarks, flag it. **This is my assumption about the shop rule — please confirm or change it.**
5. **`GEMINI_THINKING_LEVEL` is opt-in in code** (no default), so a model that does not take a level (Gemini 2.5 and older) is not broken by the upgrade. `.env.example` ships `high`.
6. **Interior ambiguity resolves to *hole, flagged*.** Staff are told rather than shown a silently wrong cutout count.

## Notes

- ⚠️ **Stored readings are keyed by image bytes**, so photos already read keep their old result under the old prompt. Use **Read again** (or clear `order_photo_reading` and `uploads/`) to pick up this change.
- ⚠️ **Restart the backend** after pulling: `GEMINI_THINKING_LEVEL` is read at startup. Your local `.env` needs it too (gitignored).
- ⚠️ **Results still vary a little run to run** on a small model. Repeat runs held after the final edits, but corner counts on the rounded-corner sheet and holes drawn inside cutout boxes remain judgement calls — which is why doubtful values are flagged.
- ⚠️ **`order_code` fix is not part of the prompt work.** The multi-tenant cutover dropped the old `DEFAULT next_business_code('ORD')`, so creating an order booking returned a 500 (`null value in column "order_code"`). The insert now supplies the code, per company and booking date. `pi_code`, `party_code`, `work_order_code` and `enquiry_code` are also `NOT NULL` with no default and no insert supplies them; **I did not test those** and expect the same failure.
- No automated test suite exists in this repo. The nine-sheet check was a throwaway script (not committed) that called the configured Gemini model without writing to the DB.
- Docs update lives in `ai-extraction-plans/`, not in this repo.

## Next Steps

- Confirm decision 4 (larger of opposite sides).
- Same `*_code` fix for party, enquiry, work order and proforma invoice — or one migration restoring the defaults.
- Keep the nine sheets as a fixture set and re-run on every prompt change; add any sheet that fails in review.
- Housekeeping cron for stored photos and readings (already flagged in `ai-extraction.service.ts`).

## Test plan

Run against the configured model (`gemini-3.5-flash-lite`, thinking `high`), read-only:

- [x] Split drawing, U-marks on the borders → 4 rows, 8 pieces, 4 cutouts each, no holes
- [x] Heading with a scribbled-out word, repeated under row 1 → one product on all 7 rows (8 mm, toughened, polish)
- [x] `2 inch Radius` written once → carried on every drawing with a corner arc
- [x] Struck-out `1120 x 850` is not output; bottom drawing is 2 holes, no cutouts; circle is 1500 x 1500
- [x] Four corner squares, "35 x 40 all cut out" → 4 cutouts each, no holes
- [x] `HN-10` sheet → cutouts on both panes; the X is a hole
- [x] Fixed pane over a door → fixed: 4 cutouts, no holes; door 915 x 1574: 2 holes, 3 cutouts (2 repeat runs)
- [x] Complex opening → 4 pieces (left pane, 12" transom, door leaf, right pane) with cutouts
- [x] Three similar drawings differing only in holes/cutouts → 3 rows (2 repeat runs); `Ø22mm` not counted as a hole
- [x] `npx tsc --noEmit` clean
- [x] Order booking insert verified in a rolled-back transaction → `ORD-20261002-101`
- [ ] `POST /order-bookings` through the running app
- [ ] A read with `GEMINI_THINKING_LEVEL` empty still works
- [ ] A read with the product list unavailable (DB error) still works
- [ ] A read on a non-Gemini provider still returns a valid reply with the leading `analysis`

## Acceptance Criteria

- Marks touching the glass border are counted as cutouts and complete interior marks as holes, on the nine sample sheets.
- Struck-out lines never appear as rows; a heading's thickness, glass type and edge work reach every row it covers.
- Pieces that differ in any hole, cutout or size are separate rows; identical pieces share one row with a quantity.
- A derived or doubtful value is listed in `unsureFields` and explained in `note`.
- Order bookings can be created again.
- No new migration, no change to the stored reading shape, and the app still starts with `AI_PROVIDER` empty.
