# 2026-11-02 - fir-status-items-accounts (backend)

## Summary

FIRs had no priority, status or internal remarks, so there was no way to track whether a FIR was still open. Purchases picked a bare product, so glass type, colour and thickness could be mixed in any way and nothing tied a purchase line to an accounting ledger. Bank details sat on the company record, one set only. The Price List report also returned every glass type as a separate table with no paging or filter.

This PR adds priority, status and internal remarks to FIRs (with status counts for tabs and filters on the register and the FIR report), adds two new Master Data lists (Accounts and Items), makes purchase lines pick an Item, moves bank details from the company record to Sales and Purchase accounts, and gives the Price List report a glass type filter and paging.

`tsc --noEmit` passes. The migration was **not run** and the endpoints were **not called live**, see "Test plan".

## What changed

| File | Change |
|---|---|
| `src/database/migrations/112_fir_and_item_account_masters.sql` (new) | One idempotent migration. Adds `priority`, `status`, `internal_remarks` to `sales.fir` with check constraints and a status index. Creates `master.account` and `master.item`. Adds `item_code` and `hsn_description` to `public.inventory_purchase_item`. Adds bank columns to `master.account` and copies each company's old bank details into a `SALES-BANK` account. Adds `glass_type_code`, `sales_account_code`, `purchase_account_code`, `color_code`, `thickness_mm` to items with foreign keys, and a unique index per combination |
| `src/modules/fir/fir.types.ts` | New `FIR_PRIORITIES`, `FIR_STATUSES` and their types. `priority`, `status`, `internalRemarks` added to the row, detail and list types. `FirPage` gets `statusCounts` |
| `src/modules/fir/fir.schema.ts` | Create takes `priority` (default MEDIUM), `status` (default OPEN) and `internalRemarks`. Update now only takes `status` and `internalRemarks`. List query takes `status` and `priority` |
| `src/modules/fir/fir.repository.ts` | New columns in the select, insert and filters. `updateFirRow` only writes status and remarks. List also counts FIRs per status (ignoring the status filter) for the tabs |
| `src/modules/fir/fir.service.ts` | Maps the new fields. Update no longer re-checks the references, since they cannot change |
| `src/modules/master/account/` (new, 6 files) | Accounts master: list, get, create, update, deactivate at `/master/accounts`. Type is SALES or PURCHASE, with optional bank name, branch, account number and IFSC |
| `src/modules/master/item/` (new, 6 files) | Items master: list (with page, status counts and KPIs), get, create, update, deactivate at `/master/items`. Checks the glass type against the Glass Types master and the Sales and Purchase accounts against their type. A raw material must have a product. Create needs colour and thickness |
| `src/modules/master/master.routes.ts` | Mounts `/accounts` and `/items` |
| `src/modules/master/lookup/lookup.datasets.ts`, `lookup.repository.ts` | New `items` and `accounts` lookup datasets so pickers can load them |
| `src/modules/inventory-purchase/inventory-purchase.schema.ts` | A purchase line takes `itemCode`. `productId` is only kept for old lines being edited. A new purchase requires `itemCode`. New `ResolvedPurchaseItem` type |
| `src/modules/inventory-purchase/inventory-purchase.service.ts` | New `resolveItems`: the item must exist, be active and be a raw material. The line takes the item's product, HSN description and HSN code (unless the user typed one). The line's glass type, colour and thickness must match the item. Used on create and update. Response lines carry `itemCode`, `itemName`, `hsnDescription` |
| `src/modules/inventory-purchase/inventory-purchase.repository.ts`, `inventory-purchase.types.ts` | Item joined into the line select. Insert writes `item_code` and `hsn_description` |
| `src/modules/master/company/company.repository.ts` | Company detail returns `bankAccounts` (active accounts with bank details). The old `bankName` to `bankIfscCode` fields now come from the first Sales account, falling back to the old company columns. Update no longer writes the bank columns |
| `src/modules/master/company/company.schema.ts`, `company.types.ts` | Bank fields removed from the update schema. New `CompanyBankAccount` type |
| `src/modules/report/fir/fir-report.schema.ts`, `fir-report.repository.ts`, `fir-report.service.ts` | FIR report rows carry priority and status, and the report can be filtered by both |
| `src/modules/report/price-list/price-list-report.schema.ts` | New `glassTypeId`, `page` and `limit` (max 200) |
| `src/modules/report/price-list/price-list-report.service.ts` | Response changed from `tables` to a flat `items` list with `totalCount`, `page`, `limit` and a `glassTypes` list for the filter. Every row now has `glassTypeId`. Without `page` and `limit` every row comes back (used by the downloads) |

## Why

The FIR register needed a way to track progress and urgency. The business also wanted purchases to use a fixed list of glass items (product, glass type, colour and thickness) with Sales and Purchase ledger accounts, and wanted more than one bank account kept against those ledgers instead of one set on the company profile. The Price List report needed to be filterable and pageable like the other reports.

The easy part was the FIR fields. The real work was the Items master and the purchase line check, because old purchases have no item and must still be editable.

## How

**FIR update is limited**: once a FIR is raised, only its status and internal remarks can change. Customer, date, details and the linked documents are fixed. Priority is set when the FIR is created.

**Status tabs**: the list returns `statusCounts` per status plus `ALL`. The counts use every filter except status, so each tab shows what it would hold if opened.

**Items are exact combinations**: an item is product + glass type + colour + thickness, kept unique by `uq_item_combination`. The item name follows its glass type's name from the Glass Types master. The glass type is stored by code, so renaming a type does not break the link.

**Purchase lines**: the line sends `itemCode`. The server fills the product, so the stock lot keeps the same `product_code` that work order stock picking, stock grouping and reports already read. `master.product` is not changed.

**Bank details**: documents (PI sheet, invoices) print the first active Sales account that has bank details. If there is none, they use the old company columns, so documents keep working.

**Kept the same on purpose**: the old `core.company` bank columns are not dropped. Old purchase lines keep their `product_code` and have an empty `item_code`.

## Decisions made

**Bank columns stay on `core.company`.** They are no longer edited, but dropping them would break any company with no Sales account. The migration copies them into a `SALES-BANK` account so nothing is lost.

**Account type is checked in the app, not the database.** The foreign keys only make sure the account exists. The service checks that the Sales account is a SALES one and the Purchase account is a PURCHASE one.

**Price List response shape changed.** `tables` is replaced by `items`. This breaks any other caller of `GET /reports/price-list`. The frontend in this task is updated to match.

**FIR update is now partial.** `PUT /firs/:id` only reads `status` and `internalRemarks`. Other fields sent by an old client are ignored.

## Notes

- Migration 112 must be run before the FIR, Items, Accounts, Purchase and Company screens work. It was written but **not run** by me.
- Existing FIRs become Medium priority and Open. Existing items keep an empty colour and thickness until someone edits them, and the purchase line check skips those two fields for them.
- The frontend uses the permission modules `master_item` and `master_account`. I did not find them seeded in the backend source or in migration 112, so check that they exist in the modules table, or the new menu entries will not show.
- Item and account codes are upper-cased by the schema.

## Next steps

- Run migration 112 on a copy of real data. The backfill matches items to glass types by name, name plus " glass", or code. Check that no item is left with an empty `glass_type_code`.
- Confirm the `master_item` and `master_account` modules and their rights are created for each role.
- Add tests for `resolveItems` (not raw material, wrong combination, inactive item) and for the limited FIR update.
- Decide whether the `core.company` bank columns can be dropped later.

## Test plan

**Automated tests (no new tests):**
- [x] TypeScript build (`npx tsc --noEmit`): no errors.
- [ ] No tests were written for the FIR changes, `resolveItems`, the Items and Accounts modules, or the Price List paging.

**Manual checks (local):**
- [ ] Run migration 112 and check the FIR, item and account tables - not done, the database was not used.
- [ ] Create a FIR, then update it and confirm only status and remarks change - not done.
- [ ] Create a purchase with an item, and with a glass type, colour or thickness that does not match - not done.
- [ ] Open the Company Profile and a PI sheet and check the bank details come from the Sales account - not done.
- [ ] Call `GET /reports/price-list` with `glassTypeId`, `page` and `limit`, and without them - not done.
