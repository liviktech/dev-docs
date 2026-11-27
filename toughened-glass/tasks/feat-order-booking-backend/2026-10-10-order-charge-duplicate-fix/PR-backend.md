# 2026-10-10 - order-charge-duplicate-fix (backend)

## Summary
Some order bookings showed the same charge more than once for a single product (for example Insurance, Logo, Packing and Polish three times each in the order preview). Those copies were also counted in the product total and in the order's CHARGES / TAX. This PR stops new duplicates from being created and cleans up the ones already saved.

**The cause was worked out from the code, not from the live data.** The fix was not run against a database, and the migration still has to be run. See "Test plan".

## What changed

| File | Change |
|---|---|
| `src/modules/order-booking/order-booking.repository.ts` | `recomputeChargesForItemInTx` now takes a per-item database lock first, and its insert ignores a charge that already exists for that item. |
| `src/database/migrations/059_order_booking_charge_unique_auto.sql` (new) | Removes existing duplicate charge rows and adds a unique index so a repeat cannot be saved again. |

## Why
Charges for an item are recalculated by deleting the item's non-manual charges and inserting them again. The booking page sends several requests at the same time, and several of them can trigger that recalculation. When two or three ran together, each one deleted the old rows before the others had committed, and then all of them inserted, so the item ended up with every charge two or three times. The table `sales.order_booking_charge` had no rule that stopped this, only plain indexes.

## How
**Lock per item**: The recalculation starts with `pg_advisory_xact_lock` on the item id. A second run waits until the first has committed, and then sees the committed rows when it deletes, so it replaces them instead of adding to them. The lock is released when the transaction ends.

**Unique index as a safety net**: The migration adds a unique index on item and charge code, only for the non-manual rows. Manual charges are free lines the user types in and may repeat, so they are left out of the index. The insert uses `ON CONFLICT ... DO NOTHING` on that index, so a missed case would skip the extra row instead of failing the whole save.

**Clean up first**: Before the index is created, the migration deletes the extra copies and keeps the oldest row for each item and charge code (non-manual rows only).

## Decisions made
**Fixed it at the database level as well as in the code.** A lock alone would stop the race, but old duplicates and any future code path would still be able to create copies. The index makes that impossible.

**Kept the oldest row when cleaning up.** The copies are identical because they come from the same calculation, so it does not matter which one stays. The oldest is the simplest rule to explain.

## Notes
- Run the migration before using Replica on a booking that may have duplicates. Replica copies charge rows straight from the source item, so with the new index it would fail on an item that still has duplicate rows.
- The order preview and the Charges tab need no change. They were showing what the API returned.
- To confirm the cause on real data, run `SELECT charge_code, COUNT(*) FROM sales.order_booking_charge WHERE order_booking_item_id = '<item id>' GROUP BY 1`. Three rows per code means the duplicates were stored. One row per code means the duplication would be coming from somewhere else.

## Next steps
- Run `npm run migrate` in the backend on each environment, then open one booking that had duplicates and check each charge shows once and the totals dropped.

## Test plan
**Automated tests (no test files cover this change):**
- [x] `npx tsc --noEmit`: no errors.
- [ ] Migration `059_order_booking_charge_unique_auto.sql`: not run. It needs to be run against a database, and no database was used for this PR.

**Manual checks (local):**
- [ ] Open a booking that showed repeated charges, run the migration, reload it, and confirm each charge appears once and the Product Total and CHARGES / TAX are lower. Not done.
- [ ] Open the same draft booking in two browser tabs at the same time and confirm no new duplicates appear. Not done.
- [ ] Replica a booking after the migration and confirm the copy has the same charges. Not done.
