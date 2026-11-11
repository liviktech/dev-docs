# 2026-10-15 - cutout-rule-hole-counts (frontend)

> **Linked PR (backend):** https://github.com/liviktech/rubine-toughened-glass-backend/pull/92

## Summary

In the "Read an order from a photo" window, the line next to the glass name added holes, centre holes and cutouts into one "hole(s)" number, so "6 hole(s)" could really be 2 holes and 4 cutouts. This PR counts the three kinds separately and shows each one, in the window and in the "Read from photo" column of the Products table. A model comparison from this task (`gemini-3.5-flash` and `gemini-3.6-flash` performed well, `gemini-3.5-flash-lite` did not) is recorded under "Notes" and in full in the backend PR.

The change was checked with the type check and lint only. It was **not** run in a browser in this PR, see "Test plan".

## What changed

| File | Change |
|---|---|
| `src/features/ai-extraction/group-rows.ts` | New `MarkCounts` type, `countMarks()` (adds up holes, centre holes and cutouts of a set of rows, a blank counts as 0) and `describeMarks()` (turns the counts into words like "2 hole(s)", "1 centre hole(s)", "3 cutout(s)", leaving out any kind that is 0). |
| `src/features/ai-extraction/photo-review-panel.tsx` | Each glass group keeps three counts instead of one `holes` total. The line next to the glass name shows each kind that is above 0, for example `4 row(s) · 4 piece(s) · 2 hole(s) · 1 centre hole(s) · 3 cutout(s)`. |
| `src/features/order-booking/draft-products.ts` | `draftHoles()` replaced by `draftMarks()`, which uses the same helper. |
| `src/features/order-booking/products-tab.tsx` | The "Read from photo" column on the Products table shows the same separate counts. When there are none it says "no holes or cutouts found" (it said "no holes found"). |

## Why

Holes, centre holes and cutouts are different things on the factory floor and are charged differently, so a single combined number hides what the AI found. The review window should let staff see at a glance which kind there is.

## How

**One shared helper**: the window and the Products table both use `countMarks()` and `describeMarks()` from `group-rows.ts`, so the two places can never disagree about how the words are made.

**Kept the same on purpose**: the counts are still the sum of each row's per piece count and are **not** multiplied by the row's quantity, exactly as before. A row with quantity 2 and 2 holes still shows 2 holes. Nothing is sent to or saved on the server by this change.

## Decisions made

**Products table wording is changed too.** The request was about the window, but the Products table summary had the same combined number, so it is fixed the same way. When there is nothing it now says "no holes or cutouts found", because cutouts are part of what it checks.

**Kinds with 0 are left out.** A group with only cutouts shows only cutouts, so the line stays short.

## Notes

- **Model comparison, short version.** The same sheet was read four times: `gemini-3.5-flash` (15 cutouts in total) and `gemini-3.6-flash` (16) performed well, and agreed on all sizes and nearly all counts. `gemini-3.5-flash-lite` did not: its two reads disagreed (8 cutouts with every pane flagged, then every cutout count blank). One sheet only, judged by the team. Full table and limits are in the backend PR of the same task.
- The feature docs were updated in the same task: `dev-docs/toughened-glass/features/ai-extraction/`.
- Lint shows 2 warnings in the changed files (`photo-review-panel.tsx` line 431 and `products-tab.tsx` line 664, both about hook dependencies). They are in lines this PR does not touch.

## Test plan

**Automated tests (no automated test suite exists in this repo):**
- [x] `npx tsc -b`: no errors in `group-rows.ts`, `photo-review-panel.tsx`, `draft-products.ts` or `products-tab.tsx`.
- [x] `npx eslint` on the four changed files: 0 errors, 2 warnings (see "Notes", not from this change).

**Manual checks (local):**
- [ ] Read a photo that has holes, centre holes and cutouts and check the window line shows each kind apart - not run in this PR, it is being checked by hand by the reviewer. Covered by the type check only.
- [ ] Check the "Read from photo" column on the Products table after adding rows, and a draft with no holes or cutouts shows "no holes or cutouts found" - not run, same reason.
