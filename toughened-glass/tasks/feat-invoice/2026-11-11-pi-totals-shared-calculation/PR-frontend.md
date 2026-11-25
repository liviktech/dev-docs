# 2026-11-11 - pi-totals-shared-calculation (frontend)

## Summary
The Proforma Invoice (PI) preview sheet and the PI view modal showed the same grand total but different Basic Amount and Round Off. This PR adds one shared calculation, `computePiTotals`, and uses it in the create form, the view modal, the preview sheet and the PDF. Basic, tax, round off and grand total now match everywhere, for both With Material and Without Material PIs. The backend is not changed. Only the type check was run (passes). It has not been checked in the browser against live data.

## What changed
| File | Change |
|---|---|
| `src/lib/pi-totals.ts` (new) | Shared `computePiTotals` and `isInterstatePi`. One place for basic, assessable, CGST/SGST/IGST, gross, round off and grand total. |
| `src/features/pi-status/components/pi-status-drawer.tsx` | The view modal now reads assessable, tax, round off and grand total from `computePiTotals`. Without Material lines show amount 0. Balance is grand total minus advance. |
| `src/features/invoices/create-proforma-invoice.tsx` | Replaced the local assessable, tax and grand total arithmetic with `computePiTotals`. The payload shape is the same. |
| `src/features/pi-status/components/proforma-pdf-data.ts` | Builds sheet and PDF data from `computePiTotals`. Without Material items get rate and amount 0. The machining row is flagged `includedInBasic`. Basic is glass plus machining. |
| `src/features/pi-status/components/proforma-invoice-sheet.tsx` | Reads basic, assessable, CGST/SGST/IGST, round off and grand total straight from the totals. Skips charge rows flagged `includedInBasic`. Removed the old processing fallback. |
| `src/lib/pdf/proforma-invoice-pdf.ts` | Same as the sheet: skips `includedInBasic` rows, removed the processing fallback, and the interstate check no longer uses the state code. |
| `src/features/pi-status/components/final-invoice-as-proforma-data.ts` | No real change. It reuses the same sheet and PDF and its behaviour is kept as before. |

## Why
The sheet and the modal used different formulas, so two numbers on screen disagreed even though the grand total was the same. Two causes were found.

**Without Material:** the backend sets the rate to 0, but the code used `Number(s.ratePerSqm) || Number(itemObj.rate) || rate`, which treats 0 as missing. It fell back to `basicAmt / area`, which gave a phantom glass value (5,412.96 in the example checked).

**With Material:** the sheet skipped charge rows named PROCESSING, MACHINING or FABRICATION and recomputed the assessable value itself, so machining was dropped. The stored grand total hid this because the round off was forced to make up the gap.

The modal's formula was the correct one, so it became the shared formula.

## How
**One formula**: basic = glass + machining. Assessable = basic + order-level charges - discount. Tax = assessable x rate (CGST/SGST split, or IGST when interstate). Gross = assessable + tax. Grand total = `Math.round(gross)`. Round off = grand total - gross, so it always stays within +/-0.50.

**Machining counted once**: the stored `basicAmount` already includes machining (the create form saves glass + fabrication). The machining row is kept in the list for display but flagged `includedInBasic`, so the sheet and PDF do not add it a second time.

**Zero rates respected**: Without Material items are forced to rate and amount 0 instead of falling back to a computed rate.

**Interstate rule**: one rule everywhere, the modal's. A PI is interstate when `cstType` contains "INTER" or `igst` is above 0. The PDF no longer guesses from the state code.

**Kept the same**: the create form payload shape, the backend, and the final invoice behaviour.

## Decisions made
**Used the modal's formula as the shared one.** The sheet's version dropped machining and relied on a forced round off, so it was not a safe base.

**Show the recomputed values after the fix.** The sheet now shows the numbers from the shared formula, not the stored ones. For an older PI whose stored total came from the old forced round off, the shown round off can differ from what was printed before.

## Test plan
**Automated tests (type check passing):**
- [x] `tsc -b --noEmit`: no errors.
- [x] Scratch run of `computePiTotals` (not committed): Without Material example gives basic 150.36, assessable 301.11, CGST/SGST 27.10 each, round off -0.31, grand total 355. With Material IGST example (glass 5,412.96) gives basic 5,563.32, assessable 5,714.07, round off +0.40, grand total 6,743.

**Manual checks (local):**
- [ ] Without Material PI: sheet and modal show the same basic, tax, round off and total. Not checked in the browser yet.
- [ ] With Material PI: same check, with machining counted once. Not checked in the browser yet.
- [ ] One older PI and one final invoice: confirm nothing looks different or broken. Not checked in the browser yet.
