# 2026-11-10 - customer-lot-order-id (backend)

## Summary

In the Customer Material list, the Order column was empty ("—") for every lot that was added by hand through Add Customer Material. The order code was only stored on lots that the system creates itself when a "Without Material" order is saved. This PR makes the customer lot list work out the order from the work orders that issued glass from the lot, when the lot has no order code of its own.

Lots that already have their own order code show it exactly as before. This was not run against the database yet, only type checked.

**Setting an order when a lot is added by hand is not included on purpose** - see "Decisions made" below. This was a choice, not a miss.

## What changed

| File | Change |
|---|---|
| `src/modules/inventory/customer-lot.repository.ts` | The lot list query joins the issue records (`inventory_movement`) to the work order item and the order item, and collects the order codes for lots that have no `order_code`. `orderCode` is now the lot's own code, or those collected codes. The search box matches the same value. |

## Why

Users checked all the lots and the order was missing on the manually added ones. The lot table has no order column filled for them, so the screen had nothing to show. The query change was small. The real work was finding a link that already exists in the data (the issue records) instead of adding new data.

## How

**Fallback only**: `COALESCE(l.order_code, ord.codes)`. If the lot has its own order code it wins and the extra join does no work for it (the join is skipped with `l.order_code IS NULL`).

**Many orders**: if glass from one lot was used for more than one order, the codes are joined with a comma, sorted and without repeats.

**Kept the same**: the response field name `orderCode`, its type (text or null), and the API route. The frontend needs no change for this.

**Search**: searching by order code now also finds lots through the collected codes, not only the lot's own code.

## Decisions made

**No Order field on Add Customer Material.** I did not add an order input to the add or edit lot flow. The order is worked out from use, so it stays in sync with the work orders. This can be added later if users want to set it up front.

## Notes

- A manually added lot that no work order has issued glass from yet still shows "—". There is no order to find until it is used.
- Type check (`tsc --noEmit`) in the backend passes.

## Next steps

- Decide if users should be able to pick an order when they add a customer lot by hand.

## Test plan

**Automated tests (no test file for this repository query, none added):**
- [x] Type check (`tsc --noEmit`): passes with no errors.

**Manual checks (local):**
- [ ] Open Customer Material and look at a manually added lot that was used in a work order - not checked live yet, the query has only been type checked. It should show the order code.
- [ ] Look at a lot created from a Without Material order - not checked live yet. It should show the same order code as before.
- [ ] Search the list by an order code - not checked live yet.
