# 2026-11-16 - capacity-dispatch-piece-pricing (backend)

## Summary
This branch (`feat/demo-fixes`, three commits on top of `dev`) groups several demo fixes in the backend. Factories had no daily capacity limit, so a plan or a same-day order could overload the plant. A dispatch plan could not be postponed or show if its pieces were ready. A Final Invoice split a product line equally across its pieces, so a small piece was billed like a big one. The production plan list had no progress or summary figures. The same email could be used by two users or two sales reps. This PR adds all of these on the backend. Only the type check and the new unit test file were run. Nothing was checked end to end on live data, and migration 140 has not been run by me.

## What changed
| File | Change |
|---|---|
| `src/database/migrations/140_proforma_delivery_date_capacity.sql` (new) | Adds `promised_delivery_date`, `delivery_type` (default `STANDARD`) and `express_surcharge` to `invoice.proforma_invoice`, plus an index on the date. Safe to run twice. Written only, not run. |
| `src/modules/production-plan/production-plan.service.ts`, `.repository.ts`, `.controller.ts`, `.routes.ts`, `.schema.ts`, `.types.ts` | Daily capacity limits per day: 25 templates, 150 cutouts, 300 holes, 250 SQM of machined glass, 400 SQM in total. Saving a plan stops with a clear message (`CAPACITY_*_EXCEEDED`) if a date goes over. New `GET /production-plans/daily-capacity` and `GET /production-plans/check-capacity`. The plan now carries a `summary` (planned and done pieces, planned and done SQM, completion percentage, pending work orders). The list takes `fromDate` and `toDate`. Each work order in the list now also carries the product lines and stages the plan covers. |
| `src/modules/proforma-invoice/proforma-invoice.service.ts`, `.repository.ts`, `.schema.ts`, `.types.ts` | Factory approval can save a promised delivery date, delivery type (standard, express, same day) and express surcharge. A date is required for factory approval. A same-day approval is checked against today's capacity and stops with `SAME_DAY_CAPACITY_EXCEEDED` if full. |
| `src/modules/work-order/work-order.service.ts` | When work orders are made for a PI that has a promised delivery date, that date is copied to the work orders. |
| `src/modules/work-order/work-order.repository.ts`, `work-order.types.ts` | Each work order item now also returns `piece_sqm` (chargeable area, else booked size area). Used by the stage progress. |
| `src/modules/production-scan/production-scan.service.ts`, `.types.ts` | Each stage in the work order progress now returns `completedSqm` and `totalSqm`. |
| `src/modules/dispatch-plan/dispatch-plan.routes.ts`, `.controller.ts`, `.service.ts`, `.schema.ts`, `.types.ts` | New `POST /dispatch-plans/:planId/postpone` (new date, one of six reasons, remarks of at least 10 characters). The plan now returns `postponements` (history) and `allLinesReady`, and each line returns `isReady`. |
| `src/modules/dispatch-plan/dispatch-plan.repository.ts` | Saves and lists postponements. A line is ready when its work order is completed, or when every live piece has finished every stage. Postponing also moves the planned date on all lines. |
| `src/modules/final-invoice/piece-shares.ts` (new) | Pure maths for a piece's share of a product line, from its own size. |
| `src/modules/final-invoice/piece-pricing.ts` (new) | Loads the size, charge and logo data that `piece-shares.ts` needs. |
| `src/modules/order-booking/charge-allocation.ts` (new) | Shares each item charge out to the sizes that cause it (hole and cutout counts, SQM, pieces, or glass amount). The shares always add up to the charge. |
| `src/modules/final-invoice/final-invoice.service.ts`, `.repository.ts` | A Final Invoice prices each piece from its own size. The eligible pieces now return their own amount, SQM and charge parts. When an old invoice is read back, the repository checks which method reproduces its stored taxable value, so old invoices keep showing what they were made with. |
| `src/modules/order-booking/order-booking.service.ts`, `.repository.ts`, `.types.ts` | The full booking returns, for each size, its share of the item charges (`chargeAllocations`, `chargesAmount`) and each charge returns its `quantitySource`. |
| `src/modules/master/user/user.service.ts`, `.repository.ts`, `.types.ts` | A user cannot be created or edited with an email another user already has (`EMAIL_TAKEN`). `company_id` is now returned on the user row for this check. |
| `src/modules/master/sales-rep/sales-rep.service.ts`, `.repository.ts` | Same rule for sales reps (`SALES_REP_EMAIL_TAKEN`). |
| `src/modules/attendance/attendance.repository.ts` | Same rule when an employee gets or changes a login email (`EMAIL_TAKEN`). |
| `src/modules/master/lookup/lookup.repository.ts` | The item lookup now returns `inProduct`, true when the item's product has an active BASE rate row for its glass type, colour and thickness. The Purchase screen uses it. |
| `src/modules/notifications/attachments/pi-sheet/pi-sheet.pdf.ts` | The emailed PI PDF uses fixed column widths so the long table fits, and the customer name is removed from the sign box. |
| `tests/final-invoice-piece-pricing.test.ts` (new) | Unit tests for the charge allocation and the piece shares. |

## Why
Capacity and same-day delivery were a demo ask: the factory needs a hard stop before more work is promised than the plant can make. Dispatch postponement and readiness came from the same demo, so the team can see why a dispatch moved and whether the glass is actually ready. The piece pricing change fixes a billing problem: one product can have a 169 SQM piece with holes and a 0.004 SQM piece with none, and the old equal split billed them the same. The email checks stop two logins or two reps sharing one address. The real work was the piece pricing, because old invoices must keep showing the same figures.

## How
**Capacity**: for each date in a plan, the repository adds what the plan brings (templates, cutouts, holes, SQM, machined SQM) to what other non-rejected plans already have on that date, ignoring the plan being edited. If any limit is passed the save stops. Same-day PI approval uses the same numbers for today in India time.

**Piece shares**: a piece's share of a product line is its size's value per piece over the total of all sizes, so the shares of all pieces of a product add up to exactly 1. The value used is the glass amount for the glass, SQM for SQM, and for each charge line the part of that charge the size causes (a hole charge follows the pieces with holes). If a piece has no size or nothing to measure, it takes an equal share, which is the old behaviour. The batch that takes the last pieces of a product still takes what is left, so the invoices of a product add up to the PI line.

**Old invoices keep their numbers**: when an invoice is read back, the code tries the per-piece way and the old equal way and keeps the one that reproduces the stored taxable value. The old way stays the default.

**Charge allocation**: hole, cutout, logo and grinding charges follow that size's own count times its pieces. SQM charges follow chargeable SQM. Quantity charges follow pieces. Fixed, percentage and manual charges follow glass amount. If every driver is zero it falls back to glass amount, then SQM, then pieces. The last size takes the rounding drift.

**Dispatch readiness**: calculated in the SQL for each line, so the list and the view modal use one rule.

**Kept the same on purpose**: the postponement table (migration 139) was already in `dev`, so only the code for it is new here. Plan approval rules are unchanged apart from a clearer message when someone approves their own plan.

## Decisions made
**Limits are written into the code.** The 25, 150, 300, 250 and 400 figures are constants in the plan service, not a setting. I did not add a settings table because the demo only needed fixed numbers. Changing them needs a code change.

**Two column mistakes fixed in the new postponement code.** While testing in the browser, the dispatch plan view returned a 500 and postponing failed, both because the new queries used `created_by` while migration 139 names the column `postponed_by`. Both queries now use `postponed_by`. These were mistakes in this branch, not old bugs.

## Notes
- `original_date` on a postponement is required, and it is taken from the first plan line. A plan whose lines have no planned date would fail on that column. This was not hit in testing.
- The three branch commits are `a758500` (email checks), `c27f98f` (piece pricing and charge allocation) and `2e24aca` (capacity). The work order SQM, the progress and the `inProduct` changes are in these commits.

## Next steps
- Confirm the progress percentage rule with the team, then make the list and the plan summary agree.
- Add tests for the capacity checks (`assertDailyCapacityLimits`, `checkCapacityForOrder`) and for dispatch readiness. Only the piece pricing has tests today.

## Test plan
**Automated tests (11 passing, new file only):**
- [x] `tests/final-invoice-piece-pricing.test.ts`: all 11 cases pass (charge allocation adds up to the charge, a fixed charge follows glass amount, zero drivers fall back, a logo makes every piece carry it, pieces of a product add up to exactly 1, any split into invoices adds up to the whole line, small pieces cost their own glass, each charge follows the pieces that cause it, a single size matches the old equal split, a piece with no size takes an equal share, an unknown charge code falls back to the item's charges).
- [x] Backend type check (`npx tsc --noEmit`): passes.
- [ ] Full backend test run - not run for this document.
- [ ] Tests for capacity limits and dispatch readiness - not written.

**Manual checks (local):**
- [x] Dispatch plan view loaded after the `postponed_by` fix was found from the server log. The fix itself was not re-run in the browser by me.
- [ ] Postponing a dispatch plan after the fix - not re-run, the user will retry.
- [ ] Migration 140 - not run by me, the team runs it.
- [ ] Capacity limits on a plan, and a same-day PI approval over the limit - not checked.
- [ ] A Final Invoice for part of a product with different sized pieces, and an old invoice read back - not checked.
- [ ] Duplicate email on user, employee and sales rep - not checked.
