# 2026-11-17 - capacity-breakdown-item-tax (backend)

## Summary
This branch (`feat/production-plan`, one commit on top of `dev`) fixes how daily capacity is counted and tightens a few rules around items and dispatch. The daily capacity count was wrong in two ways: it counted every piece of a work order even when the plan only covered part of it, and it ignored same-day orders that had no plan yet. Items could be saved without an HSN code or Tax Profile unless they were sold finished goods. A dispatch postpone changed the plan date but not the promised date on the work orders and PIs, and any user could postpone. This PR changes all of these on the backend. Only the type check was run (`npx tsc --noEmit`, no errors). The new SQL was not run against a database and nothing was checked end to end, so the capacity numbers are untested on live data.

## What changed
| File | Change |
|---|---|
| `src/modules/production-plan/production-plan.repository.ts` | Daily capacity now counts only what the plan lines cover (planned pieces or planned SQM as a share of the item), and only plans that are `SUBMITTED` or `APPROVED`. Confirmed same-day PIs with no active plan on that date are added on top. The result now also returns a `breakdown` list (one row per plan or same-day PI with its SQM, cutouts, holes and templates). `calculateLinesCapacityImpact` uses the same share logic and accepts planned unit, quantity and area. The plan list takes `fromDate` and `toDate` together and matches plans that have a line in that range, or plans with no lines created in that range (India time). |
| `src/modules/production-plan/production-plan.service.ts` | Approving a plan now re-checks the daily capacity limits first, so a plan that went over after it was submitted cannot be approved. `checkCapacityForOrder` takes optional templates, cutouts and holes and blocks a same-day order if any of them passes 25, 150 or 300. `assertDailyCapacityLimits` accepts the wider line shape. |
| `src/modules/proforma-invoice/proforma-invoice.repository.ts` | Factory approval with an express surcharge now writes it to the PI charges (add, update or remove the `DELIVERY_SURCHARGE` row) and moves the PI charge and total by the difference. `getProformaInvoiceCapacityRequirements` also returns templates, cutouts and holes, and treats grinding, polish, bevel, notch, hole, cutout or machining charges as machining. |
| `src/modules/proforma-invoice/proforma-invoice.service.ts` | The same-day capacity check now also runs when the promised date is today or earlier (not only when the type is `SAME_DAY`), and passes templates, cutouts and holes to the check. |
| `src/modules/dispatch-plan/dispatch-plan.repository.ts` | Postponing a dispatch plan now also moves `delivery_date` on the linked work orders and `promised_delivery_date` on the linked PIs. |
| `src/modules/dispatch-plan/dispatch-plan.service.ts` | Postponing is limited to Production Manager, Factory Manager, Plant Manager, Admin and Super Admin (`DISPATCH_POSTPONE_FORBIDDEN`). |
| `src/modules/dispatch-plan/dispatch-plan.controller.ts` | Passes the user's role to the postpone service. No other change. |
| `src/modules/master/item/item.schema.ts` | HSN code (4 to 8 digits) and Tax Profile are now required on create. On update they are still optional, but if sent they must be valid. |
| `src/modules/master/item/item.service.ts` | `checkSalesTaxFields` now needs an HSN code and Tax Profile on every item, not only on sold finished goods. A raw material still cannot be on the Sales account. |
| `src/modules/master/lookup/lookup.datasets.ts` | A change to the Items master now also refreshes the `products` lookup. |

## Why
The capacity meter on the production plan screen was showing wrong figures, so planners could not trust it. A plan that covered 10 of 100 pieces was counted as all 100, and a same-day order that was already promised did not count at all. The postpone fix keeps the dates the factory and the customer see in step with the dispatch plan. The item rule makes sure every item can be taxed, since the old rule left a gap for anything that was not a sold finished good. The real work was the capacity SQL, because the same share logic has to give the same numbers in the meter, the save check and the approval check.

## How
**Capacity share**: for each item on a date, the planned share is planned quantity over total pieces when the unit is pieces, or planned area over total SQM when the unit is SQM. When there is no usable quantity or area, the whole item counts. Templates, cutouts, holes and SQM are all scaled by that share. If an item has several lines on one date, the biggest share is used so it is not counted twice.

**Machining**: an item counts as machined if any of its pieces has holes, cutouts or a non-block shape, or if any line uses a work center other than cut or temper. Machined SQM is the share of those items.

**Same-day PIs**: a confirmed PI whose promised date is the target date and which is same-day (or promised for today) is added, unless its work order already has an active plan line on that date. This avoids counting the same glass twice.

**Breakdown**: built from the same two parts (plans and same-day PIs) so the list adds up to the totals. The three queries run together.

**Kept the same on purpose**: the limits (25 templates, 150 cutouts, 300 holes, 250 machined SQM, 400 total SQM) are unchanged. The response of the capacity endpoint only gains `breakdown`, nothing is removed.

## Decisions made
**Approval re-checks capacity.** I added the check at approval, not only at save. A plan can go over the limit after it is submitted because other plans are approved in between. The cost is that an approver can now get a capacity error. I did not add an override.

**Largest share when lines repeat.** When one item has several lines on the same date (for example cut and temper), I take the largest share instead of adding them. Adding would count the same pieces twice.

**Postpone role check is skipped when no role is sent.** The service only blocks when a role is present and not in the allowed list. The controller always sends the role, so this only matters for other callers of the service.

**HSN and Tax Profile are optional on update.** Create needs them. Update does not, so a partial edit of an old item is not blocked. If they are sent, they must be valid.

## Bugs found while testing (already there before, not fixed here)
**1. The express surcharge row match is loose.** The code that finds the existing surcharge row also matches `charge_name ILIKE '%Delivery Surcharge%'`, not only the basis. A user charge with that text in its name could be changed or removed by mistake. This was read in the code and not checked against real data. It is in the lines this PR adds, so it is a risk of this change and not an old bug, and it is listed here so it is not lost.

## Notes
- The type check passed with no errors. No SQL in this PR was run against a database, so the capacity queries are checked by reading only.
- No migration is in this PR. The PI delivery columns and the postponement table come from earlier migrations already in `dev`.
- Existing items with no HSN code or Tax Profile are not changed. They will be asked for the missing values the next time they are edited and sent in full.
- The matching frontend PR is `PR-frontend.md` in this folder.

## Next steps
- Add tests for `getDailyCapacityUsage` (partial plan share, same-day PI, plan plus same-day PI) and for `checkCapacityForOrder`.
- Check the capacity numbers on a copy of live data against a plan that is known to be partial.
- Decide if old items with no HSN code or Tax Profile need a one-time data fix.

## Test plan
**Automated tests (none added or run):**
- [ ] No test file was added or changed for this PR.
- [x] `npx tsc --noEmit`: no errors.

**Manual checks (local):**
- [ ] Daily capacity for a date with a plan that covers only some pieces - not run, no database check was done.
- [ ] Same-day PI approval blocked when templates, cutouts or holes pass the limit - not run.
- [ ] Approving a plan that now goes over the limit - not run.
- [ ] Postpone a dispatch plan as an allowed role and as another role - not run. Work order and PI dates were not checked after a postpone.
- [ ] Create an item without an HSN code or Tax Profile and see the error - not run against the API.
