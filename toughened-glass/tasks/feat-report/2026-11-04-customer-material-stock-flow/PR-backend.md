# 2026-11-04 - customer-material-stock-flow (backend)

## Summary

A customer lot could only be created by saving a Without Material order item, and the Work Order could only cut from the one lot tied to that item. This PR lets a customer lot exist on its own, adds endpoints to create, edit, delete and filter customer lots, and makes the Work Order stock picker and the cutting step use the customer's lots (matched by customer, product and thickness) the same way it uses normal stock.

`npx tsc --noEmit` passes. The new SQL (edit, delete, filters, work order lot match) and the three migrations were not run against a database in this session, so they are checked by type check and reading only.

**Moving existing customer lots to the new rules is not included on purpose** - see "Notes" below. This was a choice, not a miss.

## What changed

| File | Change |
|---|---|
| `src/database/migrations/120_extra_raw_sheets.sql` (new) | `extra_raw_sheets` on the order item and `raw_sheet_slot` on the lot, with a unique index on item and slot. Belongs to the earlier Add Pieces work and ships in the same commits. |
| `src/database/migrations/121_standalone_customer_lot.sql` (new) | The customer lot check now needs only a party code, so a lot no longer needs an order item. |
| `src/database/migrations/122_customer_lot_party_description.sql` (new) | `party_description` (250 chars) on `inventory_lot`. |
| `src/modules/inventory/customer-lot.repository.ts` | `createCustomerLotsInTx` (one lot per sheet row), `updateCustomerLotInTx`, `deleteCustomerLotInTx`. The list now filters by party code and a received date range, and each row also carries product code, colour, glass type, remark and party description. `assertCustomerGlassProvided` does nothing now, and `syncCustomerLotForItemInTx` no longer creates lots from an order item. |
| `src/modules/inventory/inventory.routes.ts` | New `POST /customer-material` (ADD), `PUT /customer-material/:id` (EDIT), `DELETE /customer-material/:id` (DELETE). List query accepts `partyCode`, `fromDate`, `toDate`. Sheet rows accept `partyDescription`. |
| `src/modules/inventory/inventory.controller.ts` | Handlers for create, update and delete. |
| `src/modules/inventory/inventory.service.ts` | `createCustomerMaterial`, `updateCustomerMaterial`, `deleteCustomerMaterial`. A bad customer, product, colour or location comes back as an `INVALID_REFERENCE` error. |
| `src/modules/work-order/work-order-inventory.service.ts` | A Without Material item now builds a group keyed by customer instead of one lot. The lots locked for cutting are that customer's unreturned lots with the same product, thickness, type and colour (a lot with no type or colour matches any), plus any lot picked in Select stock. Manual selections now apply to customer groups too. |
| `src/modules/work-order/work-order-stock.service.ts` | The stock candidates for a customer group come from that customer's lots, and the response includes `customerPartyCode`. Saving a selection checks the lot belongs to the right customer for a customer group. |
| `src/modules/order-booking/order-booking.repository.ts` | Saves and copies `extra_raw_sheets` on the item. |
| `src/modules/order-booking/order-booking.schema.ts` | `extraRawSheets` accepted (up to 50 rows). |
| `src/modules/order-booking/order-booking.service.ts` | Returns `extraRawSheets` on the item. |
| `src/modules/order-booking/order-booking.types.ts` | Types for `extraRawSheets`. |

## Why

The store receives the customer's glass before and apart from any one order, and one delivery may serve several orders. Tying a lot to an order item made that impossible, and it also left no way to fix a typing mistake or remove a lot added by mistake.

## How

**Create**: One lot per sheet row, with the next `LOT` business code, customer, product, thickness, colour, glass type, location, received date and the row's party description. Area is width times height times quantity.

**Edit and Delete**: Both check the lot is a customer lot of the company. The sheet size and quantity cannot change, and a lot cannot be deleted, once any stock movement exists for it or it was returned. The error is `CUSTOMER_GLASS_IN_USE`.

**Work Order stock**: For a Without Material item the group key is product, thickness, glass type, colour and customer. Candidates and the cutting engine use the same lot rules, so Select stock and Approve cut from the same lots. The booked colour is used as a filter, and a customer lot with no colour is allowed.

**Order booking**: The booking no longer needs sheet details. Old items that already have sheets keep them and their existing lots.

## Decisions made

**Customer lots are matched by customer, product and thickness, not by the order item.** This is what lets one delivery serve several orders. The cost is that two orders for the same customer and glass share a pool.

**A lot with no glass type or colour matches any type or colour.** Lots entered by hand may not have them, and a strict match would leave those lots unusable.

**Sheet changes and Delete are blocked after any stock movement, not only after cutting.** Any movement means the lot's numbers are already used by a work order, so changing them would break the ledger. Edits to the customer, glass details, location, date, remark and party description are still allowed.

## Notes

- Existing customer lots made from order items are not changed. They keep their order code and still match by customer, product and thickness. Lots from before this change that were never created (an item saved with no sheet) stay missing.
- The old `uq_inventory_lot_customer_item_slot` index is kept. Lots with no order item are not affected by it because the item id is empty.
- Migrations 120, 121 and 122 are new in this range and need `npm run migrate`.

## Next steps

- Run the migrations and test create, edit, delete and the list filters against a real database.
- Test a Work Order approval for a Without Material order, once with lots in place and once with none.

## Test plan

**Automated tests (no test cases were added or run):**
- [x] `npx tsc --noEmit`: passes with no errors.
- [ ] Unit or integration tests: none exist for this module, so none were added.

**Manual checks (local):**
- [ ] `POST`, `PUT` and `DELETE /inventory/customer-material`: not run, no database run in this session.
- [ ] List filters by customer and date: not run.
- [ ] Work Order approve and cutting from a customer lot: not run.
- [ ] `npm run migrate` for 120, 121 and 122: not run.
