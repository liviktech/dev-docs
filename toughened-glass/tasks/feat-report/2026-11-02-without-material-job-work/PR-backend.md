# 2026-11-02 - without-material-job-work (backend)

## Summary

Orders where the customer brings their own glass (Job Type "Without Material") had no real support in the system. The raw sheet size was never stored, so the cutting engine had nothing to cut from, the glass was not kept apart from Rubine's stock, and the glass was still billed as material. This PR stores the customer's raw sheet on the order item, creates one inventory lot for it, keeps that lot out of every normal stock list and report, locks the Work Order to it, lets the store mark the balance as returned, and shows the balance on the gate pass. It also fixes the final invoice, which dropped the order level charges (Admin, Wooden Box, Insurance) for these orders.

`tsc --noEmit` passes. The two new migrations (112 and 113) were **not run** from here because the `.env` points at the shared dev database. No automated tests were run and nothing was run against a database by me.

**Hiding the Material label on the proforma status drawer, proforma PDF and final invoice screens is not included on purpose** - see "Next steps" below. The amounts there are already 0 for these orders, this was a choice, not a miss.

## What changed

| File | Change |
|---|---|
| `src/database/migrations/112_without_material_customer_lot.sql` (new) | Adds `raw_sheet_width`, `raw_sheet_height`, `raw_sheet_count` to `sales.order_booking_item`. Adds `is_customer_material`, `party_code`, `order_code`, `order_booking_item_id` to `public.inventory_lot`, with a check and a unique index so each order item has at most one customer lot. |
| `src/database/migrations/113_customer_lot_return.sql` (new) | Adds `returned_at`, `returned_by` and `returned_snapshot` (what was handed back) to `public.inventory_lot`. |
| `src/modules/inventory/customer-lot.repository.ts` (new) | All customer lot logic. `isWithoutMaterial`, `assertCustomerGlassProvided`, `syncCustomerLotForItemInTx` (create, update or remove the lot with the item), `deleteCustomerLotsForItemsInTx`, `listCustomerLots`, `setCustomerLotLocation` and `returnCustomerLotInTx` (Mark Returned). |
| `src/modules/order-booking/order-booking.schema.ts` | Item schema accepts `rawSheetWidth`, `rawSheetHeight`, `rawSheetCount`. |
| `src/modules/order-booking/order-booking.types.ts` | Row and item types carry the three raw sheet fields. |
| `src/modules/order-booking/order-booking.service.ts` | Returns the raw sheet fields. Create, update and the one request save now check that a Without Material item has all three raw sheet values. |
| `src/modules/order-booking/order-booking.repository.ts` | Saves the raw sheet fields on insert, update and clone. Syncs the customer lot after an item is inserted, updated or cloned, and removes it when the item or order is deleted. A Without Material item now gets a rate of 0, and every size row is priced at 0 (`itemRate`, `priceSizeForItemInTx`), so no glass amount is billed. |
| `src/modules/work-order/work-order-inventory.service.ts` | `loadItemSpecs` joins the item's customer lot. `buildGroups` makes a separate group locked to that lot. `lockCandidateLots` returns only that lot for it, and leaves customer lots out for every other group. Manual colour choice and manual stock selection are ignored for these groups. Replacement pieces for broken glass are cut from the customer lot too. `isWithoutMaterial` now lives in the new repository file and is re-exported here. |
| `src/modules/work-order/work-order-stock.service.ts` | The Approve screen data marks these groups `customerSupplied` and offers only the item's own lot. Saving a manual selection rejects customer lots. |
| `src/modules/inventory/inventory.repository.ts` | The stock list, stock summary and ledger report leave out customer lots. |
| `src/modules/inventory/inventory.service.ts` | `assertNotCustomerLot` blocks edit, delete and Adjust Stock on a customer lot. Adds `listCustomerMaterial`, `setCustomerMaterialLocation` and `returnCustomerMaterial`. |
| `src/modules/inventory/inventory.controller.ts` | Handlers for the three new customer material calls. |
| `src/modules/inventory/inventory.routes.ts` | New routes `GET /inventory/customer-material`, `PATCH /inventory/customer-material/:id/location` and `POST /inventory/customer-material/:id/return`. |
| `src/modules/inventory/inventory.types.ts` | `InventoryLotRow` gets the optional `is_customer_material` flag. |
| `src/modules/report/inventory/inventory-report.repository.ts` | The cutting, offcut and filter queries leave out customer lots. |
| `src/database/opening-balance-correction.ts` | The one off ledger check skips customer lots, which have no receipt movement. |
| `src/modules/dispatch/dispatch.repository.ts` | The gate pass query now returns `customer_balance_glass`: for each customer lot on the invoice, the finished pieces, the uncut sheets and the cutoffs. After Mark Returned it reads the saved list instead of the live stock. |
| `src/modules/dispatch/dispatch.types.ts` | Type for `customer_balance_glass`. |
| `src/modules/final-invoice/final-invoice.repository.ts` | New `prorationShare` (by glass value, or by item count when the value is 0). `getPiChargesAndItemsTotal` also returns `itemCount`. The charges breakdown and the saved invoice detail use it. |
| `src/modules/final-invoice/final-invoice.service.ts` | Uses `prorationShare` when creating an invoice. The eligible items response now returns `piItemCount`. |
| `src/shared/gst/gst-invoice-snapshot.ts` | Same count based share when the PI items have no value. |
| `src/shared/gst/gst-invoice-source.repository.ts` | Passes `itemCount` through to the snapshot. |

## Why

Job work needs the customer's glass tracked without mixing it with Rubine's. The plan was a customer lot that is created when the order item is saved, hidden from other orders, used only by its own Work Order, and returned to the customer. Billing had to be service only. The final invoice problem was found while testing: for these orders the PI glass amount is 0, so the invoice's share of the PI charges was 0 divided by 0, which dropped Admin, Wooden Box and Insurance. An invoice of about 358 came out as 180.

## How

**Customer lot**: Saving an item with Job Type Without Material and a raw sheet creates a lot in `public.inventory_lot` with `is_customer_material = true`, the customer's `party_code` and the item id. The sheet size is stored in mm, the lot code comes from the normal `LOT` sequence. Editing the item updates the lot. Changing it to With Material removes the lot. Once cutting has started, or the lot is returned, the sheet cannot be changed.

**Stock isolation**: Every normal stock list, summary, ledger, report and Work Order candidate query now excludes customer lots. A customer lot has no purchase, so it carries no cost and never enters valuation. The ledger reconciliation skips it because it has no receipt movement.

**Work Order allocation**: The cutting engine runs the same way, but the candidate lots for these items are only the item's own lot. Offcuts belong to that lot, so they stay with the customer and never reach general stock. An old Without Material item with no customer lot still takes nothing from inventory, as before.

**No material charge**: The item rate is forced to 0 and every size under it is priced at 0 in the repository, so the PI, final invoice and gate pass amounts come out as service charges only. Switching an existing item to Without Material reprices its sizes.

**Mark Returned**: Takes the uncut sheets and cutoffs to 0, marks the cutoffs used, and saves the date, the user and the list of what was handed back on the lot. Doing it twice, or with nothing left, gives a clear error.

**Final invoice share**: The share is still the selected glass amount divided by the PI glass amount whenever that amount is above 0, so With Material invoices are unchanged. When it is 0, the share is selected items divided by PI items.

## Decisions made

**The lot is created when the item is saved.** The plan said "when the order is saved". This app has no separate submit step. The item save is the save, and the Add Product popup only builds the request, so cancelling the popup creates nothing.

**`purchase_cost` is not a column.** The plan listed it. Lots have no cost column, the cost lives on the purchase. A customer lot has no purchase, which gives the same result (no cost).

**Customer lots are hidden from Stock, not shown in a separate flag filter.** They get their own list endpoint instead. This keeps every existing stock query safe by default.

**Manual return, not automatic.** Marking returned is a deliberate click, because the glass can still be in the factory when the dispatch is approved. There is no undo yet.

**Share by item count, not by sqm.** When the PI glass value is 0 the simplest honest fallback is the number of items. For a single item order it gives 100 percent. It is only approximate for a split invoice of items with very different sizes.

## Bugs found while testing (already there before, not fixed here)

**1. Party Approval collects the balance on its own.** In `final-invoice.service.ts`, approving as the party sets Amount Collected to the whole remaining balance, so an invoice shows "Paid in Full" even with no payment proof attached. This is existing behaviour and was **not changed** because it affects every invoice. It showed up on FI-136, where only the advance had been paid.

## Notes

- The new customer lot routes use `requireAuth` only, the same as the other inventory routes. The Control right is checked in the frontend for the buttons.
- Old final invoices saved before this fix keep their stored total. Their detail view is worked out again from the new share, so it may no longer match the stored total. Those invoices should be cancelled and created again.
- Nothing was run against the dev database. Migrations 112 and 113 still need `npm run migrate`.

## Next steps

- Hide the Material label and totals on the PI status drawer, proforma sheet and PDF, and final invoice screens for Without Material orders. Amounts are already 0 there.
- Decide the Party Approval behaviour above and change it if payments should be recorded separately.
- Save the dispatch or gate pass number, the receiver and a remark when the balance is marked returned. Add a reopen option if a lot is marked by mistake.

## Test plan

**Automated tests (none run):**
- [x] `npx tsc --noEmit` in the backend: passes with no errors
- [ ] `npm test`: not run from here. There are no tests yet for `customer-lot.repository.ts` or `prorationShare`. A test for `prorationShare` (value above 0, value 0 with counts, zero items) would be the first one to add.

**Manual checks (local):**
- [ ] Run `npm run migrate` for 112 and 113: not run, the `.env` points at the shared dev database
- [ ] Save a Without Material item, see one customer lot created, then edit its sheet size and delete the item: not run
- [ ] Approve a Work Order for it with zero Rubine stock and see it cut from the customer lot only: not run
- [ ] Complete cutting, then Mark Returned, then open the gate pass and see the finished and balance glass: not run
- [ ] Create a final invoice for a Without Material order and see Admin, Wooden Box and Insurance carried over, total close to the PI total: not run. The earlier wrong figures (FI-136 at 182 and a 90 collect amount) came from the user's own screens
- [ ] A With Material order still creates a final invoice with the same total as before: not run
