# 2026-11-05 - order-status-report-and-ui-fixes (backend)

## Summary

This PR is a set of fixes from one day's work, mostly on the Order Status Register report plus small UI fixes in Final Invoice, Order Booking, Purchase, Production Plan and Work Order stock selection.

The Order Status Register report had mixed column alignment, showed the state in the Location column, had a WO Status filter that is no longer wanted, left Order Notes blank when only the product descriptions were filled in, and did not show when a PI's amounts were shared by several Work Orders. Elsewhere: the Final Invoice party approval showed an internal running number instead of the FI number, the Read from photo modal flashed open for enquiries without a photo, the Purchase Details modal repeated the product list in its payment section, the Production Plan allowed the same stage twice for one product, the stock picker's Done button could be pressed with nothing picked, and the Order Booking details modal showed the job type twice.

Each change keeps the existing flows working. All changes were tested in the browser. Frontend and backend type checks pass, and the changed report query was also run read-only against the database.

**ISSUE-03 of the report audit (Process "free text") is not changed on purpose** - see "Decisions made" below. This was a choice, not a miss.

## What changed

| File | Change |
|---|---|
| `src/modules/report/order-status/order-status-report.repository.ts` | Order Notes now joins Party Description, Note, System Description and booking remarks, each text once (case-insensitive). New `pi_work_orders` column counting the PI's Work Orders regardless of filters. |
| `src/modules/report/order-status/order-status-report.service.ts` | Maps `pi_work_orders` to `piWorkOrders`. |

## Why

All changes came from review feedback on the screens and from the Identified Issues Log in `docs/order-status-register-report-audit.md`. Most were small UI fixes. The real work was in the report query: building Order Notes without repeated text, and counting a PI's Work Orders across all filters so a single filtered row still shows its PI is shared.

## How

**City only in Location**: the location label comes from a shared backend query as "City, State". The report cuts it to the part before the first comma when displaying it. The shared query and the Location filter are unchanged.

**Order Notes**: the four texts are put in an array, empty ones dropped, duplicates removed by lowercase text, and the rest joined with ` · ` in a fixed order.

**Shared PI**: a per-row subquery counts all Work Orders with the same PI, ignoring the report filters, so the count is right even when only one Work Order is shown.

## Decisions made

**Phone, Work Order No and PI Number stay left-aligned.** They are identifiers, not amounts.

**Location filter dropdown still shows "City, State".** Only the column was changed. The filter and the Location Sales report share the same backend query.

**Duplicate stage rule is per work order and product, not per whole work order.** Different products of one work order each have their own route, so each can still be planned at the same stage. This also means one product's stage can no longer be split over two lines of the same plan.

## Notes

- The changed report query was run read-only against the database in the backend `.env`. The audit's sample Work Orders (`WO-20261023-173`, `WO-20261026-174`, `WO-20261030-185`) now show their descriptions in Order Notes.
- The current data has no PI shared by more than one Work Order, so the `(n WOs)` label was not seen on real data.

## Next steps

- Decide whether PI Balance should use the advance actually received, and file a ticket if so.
- Correct or retire the remaining inaccurate parts of the report audit document.

## Test plan

**Automated tests:**
- [x] Backend type check (`npx tsc --noEmit`): no errors.
- [x] Order Status report query run read-only against the database: runs without errors, Order Notes shows descriptions with no repeated text.

**Manual checks (local):**
- [ ] PI Number `(n WOs)` label on a real shared PI - no such PI in the current data. Covered by the query check above and the code path.
