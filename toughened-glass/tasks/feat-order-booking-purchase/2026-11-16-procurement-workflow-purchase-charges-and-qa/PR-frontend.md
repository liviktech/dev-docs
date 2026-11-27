# 2026-11-16 - procurement-workflow-purchase-charges-and-qa (frontend)

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

**Procurement tabs (Indent Request and Purchase Request)**

| File | Change |
|---|---|
| `src/features/procurement/types.ts` (new) | Types for indents, purchase requests, list pages and the purchase's link back to its request |
| `src/features/procurement/api.ts` (new) | API calls for indents and purchase requests |
| `src/features/procurement/queries.ts` (new) | React Query hooks. Saving or changing any document refreshes all procurement lists so numbers and statuses stay in step |
| `src/features/procurement/procurement-list-state.ts` (new) | Search, status tab, date range, page and sort state for the lists, with the search sent to the server once typing pauses |
| `src/features/procurement/procurement-ui.tsx` (new) | Shared status labels, the filter toolbar (status tabs, search, dates, reset) and the General Details field layout |
| `src/features/procurement/use-item-labels.ts` (new) | Names an item line the same way everywhere: the product on its own, and "Glass Type · Colour · Thickness" beside it |
| `src/features/procurement/indent-dialog.tsx` (new) | New / Edit Indent. Product dropdown then item dropdown, width, height, quantity, UOM (always Sheets), item remarks. Same Add Row behaviour as New Purchase (one open row at a time, tick to confirm, cross to cancel, Edit, Delete with a confirmation, Enter confirms). Request Date, Required Date and Remarks on one row. The Indent Number field is commented out because the number is generated on save |
| `src/features/procurement/purchase-request-dialog.tsx` (new) | New / Edit Purchase Request. Opens with the indent's products, sizes and quantities. Quantity and remarks can change, products can be removed (with a delete confirmation) and added back, nothing new can be added. The Purchase Request Number field is commented out |
| `src/features/procurement/procurement-views.tsx` (new) | Read-only view of an indent and of a purchase request, with the Create Purchase Request / Accept / Create Purchase actions. Product and Glass Type / Colour / Thickness are separate columns. When opened from a link in the other list, the action buttons are left out |
| `src/features/procurement/procurement-tabs.tsx` (new) | The Indent Request and Purchase Request lists. Indent columns: Indent Number, Purchase Request Number, Request Date, Required Date, Products, Products (Count), Total Quantity, Total Area (SQM), Status, Created By, Created Date. The numbers are right-aligned and the text left-aligned. The Purchase Request Number (Indent list) and Indent Number (Purchase Request list) are links. The kebab View item is commented out (the number link opens the same view). Status tabs are Open / PR Created / All for indents and Pending / Accepted / All for purchase requests |

**Inventory page and Purchase screens**

| File | Change |
|---|---|
| `src/routes/inventory.tsx` | Adds the Indent Request and Purchase Request tabs after Purchase, and the New Indent header button. The Purchase tab keeps its place |
| `src/features/inventory/purchase-tab.tsx` | New Purchase Request Number column |
| `src/features/inventory/purchase-queries.ts` | Creating a purchase also refreshes the procurement lists, so the request shows as used |
| `src/features/inventory/purchase-types.ts` | Types for the Purchase Request reference, itemised charges and the QA fields on each line |
| `src/features/inventory/components/purchase-dialog.tsx` | (1) Opens prefilled from a Purchase Request (item, size, quantity, remark) and saves with the request reference; shows a Purchase Request Number field. (2) New Additional Charges section. The old single Additional Charges input is commented out and the summary strip shows the total as read-only. (3) QA columns: Received, Rejected, Accepted, QA Remark. Received follows the quantity until QA is typed, and the QA cells are read-only in correction-request mode |
| `src/features/inventory/components/purchase-view-dialog.tsx` | Shows the Indent and Purchase Request Number, the charge lines, and the QA columns (with totals) |
| `src/features/inventory/components/purchase-reference-cards.tsx` | The Payment modal's Purchase Summary lists each charge on its own line, then the total |
| `src/features/inventory/components/add-to-stock-dialog.tsx` | The amount breakdown lists each charge too |
| `src/features/inventory/components/purchase-edit-requests-dialog.tsx` | A charge-list change shows as a readable before and after for the reviewer |
| `src/features/inventory/purchase-edit-requests.ts` | Request type carries the whole charge list |

**Purchase Charges master**

| File | Change |
|---|---|
| `src/features/purchase-charges/types.ts`, `api.ts`, `queries.ts` (new) | Types, API calls and hooks for the master |
| `src/features/purchase-charges/purchase-charge-dialog.tsx` (new) | Add / Edit Purchase Charge (code, name) |
| `src/routes/admin.purchase-charges.tsx` (new) | The Purchase Charges screen: list, KPI cards, search, Active / Inactive / All tabs, status toggle, view details, deactivate with a confirmation |
| `src/components/layout/nav-items.ts`, `src/features/auth/accessible-routes.ts` | Adds Purchase Charges under Billing & Finance |
| `src/lib/lookup.ts` | New `purchaseChargeTypes` dropdown list, refreshed when the master changes |
| `src/routeTree.gen.ts` | Regenerated by the build for the new route |

## Why

The business wanted purchases to start from an internal request instead of being typed directly, to record the extra costs on a purchase (energy, diesel and similar) as separate lines they can see, and to check quality before stock is added. Today, rejected or damaged sheets could only be handled outside the system.

The easy part was the screens that look like the existing ones. The real work was keeping the existing purchase behaviour intact while adding all this: one indent to one request to one purchase had to be enforced in the database and in the service, the purchase total had to stay the one number every report and the payment screen already used, and QA had to change what goes into stock without changing what is owed to the supplier.

## How

**Purchase made from a request**: The New Purchase modal is the same modal as before. It opens with the request's products, sizes and quantities and saves the request reference. The server only allows products that the request holds, so a purchase cannot drift away from its request.

**Charges and the total**: Charges are stored as rows (type, amount, remark). The existing `additional_charges` column on the purchase header is kept and now holds the sum of the rows. Every place that already adds that column (the purchase list, the Payment modal, the stock tab, the Tally voucher amount, the payment limit) keeps working with no change. A purchase that only has the older single amount keeps it until someone adds charges, and saving charges then replaces it.

**QA and stock**: QA is stored per line. The purchase total and the payment stay based on the ordered quantity because that is the supplier's invoice. Add to Stock posts only the accepted sheets. A line saved before QA has empty QA columns, which are read as "all received and accepted", so older purchases, lots and payments are unchanged.

**Edit requests**: Charge changes use the same request and admin approval as other corrections. The list is compared entry by entry, so an unchanged list is reported as "nothing changed". The old rule that the corrected total cannot be below what is already paid still applies.

**Master and lookup**: Purchase Charges follow the same pattern as the other masters and appear in the central dropdown list, which is cleared when the master changes.

## Decisions made

**GRN was built and then removed.** A Goods Receipt Note step (partial deliveries, accepted and rejected quantities, stock posting) was built first. It was removed completely because the business does not need it. QA on the New Purchase modal replaces it, and the one purchase is received once. Migration 139 still creates the GRN tables and 143 drops them, so databases that ran 139 end up clean. Any GRN data in a shared database is lost when 143 runs.

**The Accept step on indents was removed.** An accepted indent used to be the gate for a Purchase Request. Now a saved indent goes straight to a request. The indent is locked once a request exists, and its stage (Open / PR Created) is worked out, not stored.

**Tax on additional charges was added and then removed.** A Tax % per charge was built first. It was removed, so charges are plain amounts. Migration 143 takes any charge tax off the affected purchases and their invoice records. Purchases that never had taxed charges are not touched.

**One receipt, no partial deliveries.** QA is entered when the purchase is created. A purchase is added to stock once. Rejected sheets are recorded and not tracked for replacement.

**Quantities are in sheets and the unit is fixed.** The Items master has no unit of measure, so every document shows "Sheets".

**Charges are a managed master, not a fixed list.** The user does not yet know all the charges that will apply, so users add their own under Master Data.

**Existing charge amounts are not backfilled.** Older purchases keep their single amount, shown with a note, and are not converted into rows.

**Permissions.** The new tabs use the existing Inventory rights, and the Purchase Charges screen uses the single Master Data right like the other masters. No new roles were added.

**The mobile app was not changed.** There are no mobile screens for the new tabs or the master, and the mobile purchase form still sends one lump charge amount and no QA. The server treats that as "no itemised charges" and "all accepted".

## Bugs found while testing (already there before, not fixed here)

**1. Two existing tests fail.** In `tests/inventory-purchase-create.test.ts`, the cases "accepts a complete purchase and defaults the discount and charges to 0" and "ignores totals sent by the client" fail because their sample purchase has no item, and items have been required for a while. I only updated the one assertion about additional charges; I did not fix the sample.

**2. Existing lint errors.** Files such as `src/routes/inventory.tsx` and `src/features/inventory/components/purchase-view-dialog.tsx` report errors that were already there (components exported with other items, state set inside an effect). The new Purchase Charges page shows the same "only exports components" error that the existing Charges page shows.

## Notes

- Migration 143 is destructive in one place: it drops the GRN tables. It also changes the totals of any purchase that had taxed charge rows.
- An indent opened for edit shows its rows already confirmed. A new indent starts with no rows.
- The Tally voucher amount uses the purchase header total, so it follows the new charge total. I did not trace re-syncing a purchase that was already posted to Tally.
- Several other files in both repos show as modified today (for example the Tally, AI extraction, Sales Rep and Enquiry files). Those are not part of this work and are not described here.

## Next steps

- Add mobile screens for Indent Request, Purchase Request and Purchase Charges, and update the mobile purchase form for itemised charges and QA.
- Fix the sample purchase in `tests/inventory-purchase-create.test.ts`, and add tests for charges, QA and Add to Stock with rejected sheets.
- Check how a changed total re-syncs a purchase that is already posted to Tally.
- Decide whether QA should be correctable after payment or stock (today it is only entered when the purchase is created).

## Test plan

**Automated tests:**
- [x] Frontend build (`npm run build`, which runs `vite build` and `tsc -b`): builds with no type errors. The route tree was regenerated.
- [x] ESLint on the new and changed procurement, purchase-charges, purchase dialog and purchase tab files: no new errors. The only remaining messages are the existing ones listed above.
- [ ] `tests/inventory-purchase-create.test.ts`: 10 of 12 cases pass. The 2 failures are the sample-purchase problem in "Bugs found" (they were failing before this change). I updated the additional-charges assertion but did not fix the sample, so those 2 still fail.

**Manual checks (local):**
- [x] Browser: the author tested all the screens and flows in this PR (Indent Request, Purchase Request, Create Purchase from a request, the number links, Purchase Charges master, itemised charges in New Purchase, Payment and View, edit requests, and QA with Add to Stock).
- [x] Same local database, after tax was removed: a ₹10,000 purchase with one ₹500 charge gave a total of ₹10,500. An unknown charge type and an inactive charge type were refused on a new purchase.
- [x] Same local database, run earlier while charges still had tax (not re-run after tax was removed): the list total, the invoice record and the payment breakdown all matched the purchase total; editing without sending charges kept them, and sending charges replaced them; an older single-amount purchase kept its amount. An edit request that changed charges was not applied until approved, then gave the new total; clearing the charges returned to the old total; a request with no change was refused.
- [x] Same local database: QA of 100 ordered, 90 received, 10 rejected added 80 sheets to stock; a second Add to Stock was refused; a purchase where every sheet was rejected could not be added to stock; changing the quantity of a QA-affected line through a request was refused; a purchase made from a request could be added to stock.
- [ ] Mobile app: not checked, because the mobile screens were not changed.
