# 2026-11-16 - procurement-workflow-purchase-charges-and-qa (backend)

## Summary

Before this change, a purchase could only be entered directly as a supplier Purchase Entry. There was no internal request step in front of it, no way to record extra costs such as energy or diesel as separate lines, and no quality check (QA) before stock was added.

This PR adds three things to Inventory, across the frontend and backend:

**1. A procurement flow in front of the existing Purchase Entry**
1. **Indent Request tab.** A user raises an indent for products from the Items master (product first, then the matching glass type / colour / thickness), with the sheet width, height and quantity. A saved indent can be edited until a Purchase Request is made from it.
2. **Purchase Request tab.** A Purchase Request is created manually from an indent. It starts with the indent's products and quantities. The user can change quantities and remarks, remove products (and add them back), but cannot add products the indent does not have. One indent gives one Purchase Request. A Purchase Request goes Pending to Accepted, and acceptance locks it.
3. **Create Purchase** on an accepted Purchase Request opens the existing New Purchase modal with the products, sizes and quantities filled in. The saved purchase keeps the Purchase Request reference. One Purchase Request gives one Purchase Entry, and the purchase can only contain products from its request.
4. **Links between documents.** The Purchase Request Number in the Indent table opens that request. The Indent Number in the Purchase Request table opens that indent. The Purchase table has a Purchase Request Number column.

**2. Additional charges on a purchase**
5. **Purchase Charges master (new screen).** Users configure the charges themselves (for example Energy, Diesel, Freight) under Master Data > Billing & Finance > Purchase Charges.
6. **Itemised charges in New Purchase.** A new "Additional Charges" section (Add Charge, tick to confirm, Edit, Delete with a confirmation popup) replaces the single Additional Charges amount. The Payment modal and the Purchase View show each charge on its own line. Charges carry no tax.
7. **Edit requests.** After payment or stock, charge changes go through the existing edit request with admin approval.

**3. QA on the New Purchase modal**
8. Each product line has Received, Rejected, Accepted (worked out) and a QA Remark. Only accepted sheets go into stock when Add to Stock is used.

All changes were tested in the browser by the author. Both projects compile. A few backend flows were also run against a throwaway local database (see "Test plan").

**GRN, an Accept step on indents, and tax on charges are not included on purpose** - all three were built during the day and then removed. See "Decisions made" below. This was a choice, not a miss.

## What changed

**Database**

| File | Change |
|---|---|
| `src/database/migrations/139_procurement_workflow.sql` (new) | Tables for indents and their items, purchase requests and their items, and a `purchase_request_id` on the purchase. Unique indexes make one indent give one request and one request give one purchase. It also created GRN tables, which migration 143 drops |
| `src/database/migrations/140_procurement_item_sheet_size.sql` (new) | Sheet width and height on indent and purchase request lines. The same item can be indented again in another size |
| `src/database/migrations/141_indent_remove_accept.sql` (new) | Drops the indent status and acceptance columns. An indent is now Open or PR Created, worked out from whether a request exists |
| `src/database/migrations/142_purchase_additional_charges.sql` (new) | The `master.purchase_charge_type` table, the `inventory_purchase_charge` table, and a tax column on the purchase header (dropped again by 143) |
| `src/database/migrations/143_remove_grn_charge_tax_add_purchase_qa.sql` (new) | Drops the GRN tables. Removes charge tax: the tax columns go and a purchase that had taxed charges has the tax taken off its additional charges and invoice record. Adds the QA columns (received, accepted, rejected, remark) with a check that received equals accepted plus rejected and is not more than the ordered quantity |

**Procurement module (new)**

| File | Change |
|---|---|
| `src/modules/procurement/procurement.schema.ts`, `procurement.types.ts` (new) | Validation and types for indents and purchase requests |
| `src/modules/procurement/procurement.repository.ts` (new) | Raw SQL: paged lists with search, filters and sorting; details; numbers from the existing business-code sequence (IND, PRQ); total quantity, total area and product names for the indent list |
| `src/modules/procurement/procurement.service.ts` (new) | Create and edit an indent; create a purchase request from an indent (indent row locked, one per indent); edit and accept a purchase request; checks used when a purchase is made from a request |
| `src/modules/procurement/procurement.controller.ts`, `procurement.routes.ts` (new) | Endpoints under `/procurement`. They use the Inventory rights (view, add, edit, approve) |
| `src/api/route-mapper.ts` | Mounts `/procurement` |

**Purchase module**

| File | Change |
|---|---|
| `src/modules/inventory-purchase/inventory-purchase.schema.ts` | Accepts `purchaseRequestId`, an itemised `charges` list, and QA fields per line (received, accepted, rejected, remark). QA is all or nothing, and left out means every sheet is accepted |
| `src/modules/inventory-purchase/inventory-purchase.repository.ts` | Saves the request reference, the charge rows, the header charges total and the QA columns. "Sheets still to add to stock" now means accepted sheets not yet posted |
| `src/modules/inventory-purchase/inventory-purchase.service.ts` | A purchase made from a request locks and checks that request and only allows its products. Charges are validated and totalled on the server. Add to Stock posts only the accepted sheets and is refused when nothing was accepted. Purchase details return the charges, QA and the request reference |
| `src/modules/inventory-purchase/inventory-purchase.types.ts` | Types for charges, QA and the request reference |
| `src/modules/inventory-purchase/purchase-charges.ts` (new) | Checks each charge against the master (an inactive type already on the purchase stays valid) and totals them |
| `src/modules/inventory-purchase/purchase-edit-request.service.ts`, `purchase-edit-request.repository.ts`, `purchase-edit-request.schema.ts` | Edit requests can carry the whole charge list. A line with rejected or short sheets cannot have its item, size, quantity or location corrected. A corrected line's QA follows its quantity |

**Purchase Charges master and lookup**

| File | Change |
|---|---|
| `src/modules/master/purchase-charge-type/` (new: `.schema`, `.types`, `.repository`, `.service`, `.controller`, `.routes`) | List, view, create, edit and deactivate purchase charges |
| `src/modules/master/master.routes.ts` | Mounts `/master/purchase-charge-types` |
| `src/modules/master/lookup/lookup.datasets.ts`, `lookup.repository.ts` | Adds the charge list to the central dropdown lookup and clears its cache on writes |

**Tests**

| File | Change |
|---|---|
| `tests/inventory-purchase-create.test.ts` | One assertion updated: additional charges left out now means "not sent", not 0 |

## Why

The business wanted purchases to start from an internal request instead of being typed directly, to record the extra costs on a purchase (energy, diesel and similar) as separate lines they can see, and to check quality before stock is added. Today, rejected or damaged sheets could only be handled outside the system.

The easy part was the screens that look like the existing ones. The real work was keeping the existing purchase behaviour intact while adding all this: one indent to one request to one purchase had to be enforced in the database and in the service, the purchase total had to stay the one number every report and the payment screen already used, and QA had to change what goes into stock without changing what is owed to the supplier.

## How

**Document chain**: An indent has no accept step. Once saved it can be turned into one Purchase Request, and from then on it is locked. A Purchase Request is accepted, which locks it, and then one Purchase Entry can be made from it. The unique indexes in the database back up the checks in the service, so two people clicking at once cannot create a second request or a second purchase.

**Purchase made from a request**: The New Purchase modal is the same modal as before. It opens with the request's products, sizes and quantities and saves the request reference. The server only allows products that the request holds, so a purchase cannot drift away from its request.

**Edit requests**: Charge changes use the same request and admin approval as other corrections. The list is compared entry by entry, so an unchanged list is reported as "nothing changed". The old rule that the corrected total cannot be below what is already paid still applies.

## Decisions made

**The Accept step on indents was removed.** An accepted indent used to be the gate for a Purchase Request. Now a saved indent goes straight to a request. The indent is locked once a request exists, and its stage (Open / PR Created) is worked out, not stored.

**Tax on additional charges was added and then removed.** A Tax % per charge was built first. It was removed, so charges are plain amounts. Migration 143 takes any charge tax off the affected purchases and their invoice records. Purchases that never had taxed charges are not touched.

**One receipt, no partial deliveries.** QA is entered when the purchase is created. A purchase is added to stock once. Rejected sheets are recorded and not tracked for replacement.

**Quantities are in sheets and the unit is fixed.** The Items master has no unit of measure, so every document shows "Sheets".

**Charges are a managed master, not a fixed list.** The user does not yet know all the charges that will apply, so users add their own under Master Data.

**Existing charge amounts are not backfilled.** Older purchases keep their single amount, shown with a note, and are not converted into rows.

## Bugs found while testing (already there before, not fixed here)

**1. Migration 135 fails on a fresh database.** `135_designation_master.sql` stops with "there is no unique constraint matching given keys for referenced table designation". It was already there, and it stops anyone building a new database by running every migration from the start. It does not affect the existing database. I worked around it only on a throwaway local database.

## Notes

- Run the migrations in order: `139`, `140`, `141`, `142`, `143`. They have not been run on the shared Neon database from this checkout. That database already has a `138_enquiry_confirmation_date.sql` that is not in this repo, which is why the new files start at 139.
- Migration 143 is destructive in one place: it drops the GRN tables. It also changes the totals of any purchase that had taxed charge rows.
- The backend `.env` points at the shared remote database. The backend checks below were run on a throwaway local Postgres with my own scripts. Those scripts are not committed.
- An indent opened for edit shows its rows already confirmed. A new indent starts with no rows.
- The Tally voucher amount uses the purchase header total, so it follows the new charge total. I did not trace re-syncing a purchase that was already posted to Tally.

## Next steps

- Fix the sample purchase in `tests/inventory-purchase-create.test.ts`, and add tests for charges, QA and Add to Stock with rejected sheets.
- Check how a changed total re-syncs a purchase that is already posted to Tally.
- File tickets for the bugs above (migration 135 and the two failing tests).

## Test plan

**Automated tests:**
- [x] Backend type check (`tsc --noEmit`): no errors.
- [ ] `tests/inventory-purchase-create.test.ts`: 10 of 12 cases pass. The 2 failures are the sample-purchase problem in "Bugs found" (they were failing before this change). I updated the additional-charges assertion but did not fix the sample, so those 2 still fail.

**Manual checks (local):**
- [x] Throwaway local Postgres with the full migration set (the migration 135 failure worked around): migrations 139 to 143 applied cleanly.
- [x] Same local database, service-level scripts: indent to purchase request to purchase worked; a second request for the same indent and a second purchase for the same request were refused; a purchase with a product outside its request was refused; editing an indent after its request existed was refused.
- [x] Same local database, after tax was removed: a ₹10,000 purchase with one ₹500 charge gave a total of ₹10,500. An unknown charge type and an inactive charge type were refused on a new purchase.
- [x] Same local database, run earlier while charges still had tax (not re-run after tax was removed): the list total, the invoice record and the payment breakdown all matched the purchase total; editing without sending charges kept them, and sending charges replaced them; an older single-amount purchase kept its amount. An edit request that changed charges was not applied until approved, then gave the new total; clearing the charges returned to the old total; a request with no change was refused.
- [x] Same local database: QA of 100 ordered, 90 received, 10 rejected added 80 sheets to stock; a second Add to Stock was refused; a purchase where every sheet was rejected could not be added to stock; changing the quantity of a QA-affected line through a request was refused; a purchase made from a request could be added to stock.
- [x] Same local database: a simulated purchase with taxed charge rows had its total and invoice record restored by migration 143.
- [ ] Shared Neon database: the migrations were not run there, so nothing was checked on it.
