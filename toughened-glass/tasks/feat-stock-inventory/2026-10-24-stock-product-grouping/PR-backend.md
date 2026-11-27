# 2026-10-24 - stock-product-grouping (backend)

## Summary

Stock showed one row per purchase, so a purchase with 10 mm and 12 mm glass was one row and the same glass bought twice was two rows. This PR makes a stock row a product, told apart by product, glass type, colour and thickness (glass size does not matter). Every purchase line becomes its own lot, with its own lot code, inside the product it belongs to. A product gets a fixed, readable code built from its parts (for example `TG-CSB-10-CLR`) instead of a dated number. Work orders, auto select (oldest first, cutoffs first) and the approve flow work on lots exactly as before.

**Editing the product code by hand is not included on purpose** - see "Decisions made" below. This was a choice, not a miss.

## What changed

| File | Change |
|---|---|
| `src/database/migrations/085_inventory_stock_product.sql` (new) | Adds `inventory_product_key()`. Reuses `inventory_lot_group` as the product row (adds `product_key` and `stock_code`, `purchase_id` and `lot_code` become optional). Gives lots that shared one purchase code a code of their own. Regroups every existing lot by product. Cancels a second pending edit request that now lands on the same product and points the others at the new product. |
| `src/database/migrations/086_inventory_stock_code.sql` (new) | Adds `inventory_code_abbr()` and `inventory_stock_code_base()` and recodes the existing products to the readable code. 085 was already applied on the dev database with the old dated codes, so this is a new file and not an edit of 085. |
| `src/modules/inventory/inventory.repository.ts` | The Stock list groups by product and returns the product code, type, thickness and each lot's code and received date. New `syncLotToProduct` (finds or creates the product, builds the code, moves the lot, removes a product left empty) and `deleteGroupIfEmpty`. `createLot` and `deleteLot` use them. Search also matches the product code. |
| `src/modules/inventory/inventory.service.ts` | Table rows carry type, thickness and per lot code and date. `updateLotInTx` re-syncs the product after an edit, so changing type, colour or thickness moves the lot. Deleting a product deletes its lots, only if none has history or a purchase link. Lot history for a product adds lot code, purchase number, date and supplier per lot. |
| `src/modules/inventory/inventory.types.ts` | `InventoryLotHistoryItem` gets `lotCode`, `purchaseNo`, `receivedDate`, `supplierName`. Comments say "stock product". |
| `src/modules/inventory/inventory-edit-request.service.ts` | `editLotAsAdmin` still returns a row when every lot of the product moved away (the product no longer exists, so it answers with the lot's new product). Comments updated. |
| `src/modules/inventory/inventory-edit-request.repository.ts`, `inventory-edit-request.schema.ts` | Comments only, no real change. |
| `src/modules/inventory-purchase/inventory-purchase.service.ts` | Approving a purchase no longer creates a lot group per purchase. Each line creates its own lot (the database default gives the `LOT-` code) and the repository puts it into its product. |
| `src/modules/inventory-purchase/inventory-purchase.repository.ts` | Removed `findOrCreateLotGroup` and `attachToLotGroup` (no longer used). |

## Why

Stock should be read by product, because that is how the glass is cut and used. The purchase is only where a lot came from, and it is still shown per lot. The readable code means the same product always has the same code and people can recognise it without opening the row.

## How

**Product identity**: `inventory_product_key()` lowercases product, glass type, colour code (or the colour text when there is no code) and thickness (rounded to 3 places, so 10 and 10.000 match). One product row exists per company and key, enforced by a unique index.

**Product code**: `{product}-{glass type}-{thickness}-{colour}`, with each part shortened (several words give their initials, one word keeps its first letter and consonants, a word of up to 3 letters stays). A part that is empty is left out. If another product already has the same code, the next free suffix is used (`-2`, `-3`). The code is made once when the product is created and never changes after that, even if a lot moves.

**Lots**: a lot keeps its sheets, cutoffs, movements and work order allocation. Only its `group_id` changes. Edit requests are still keyed by the row id, which is now the product id.

**Kept the same on purpose**: the work order stock service, the movements repository (`lot_id = X OR group_id = X` still works) and the edit request repository. Lot codes are now unique per lot, which the work order screens already assumed.

## Decisions made

**Product code is not editable by the user.** A user defined code was discussed and left for later. The system builds it, which keeps it unique and the same for the same product.

**Code change went into a new migration 086.** 085 had already been applied on the dev database with the old dated codes, so the readable code is a new file and 085 was put back as applied.

**Lots that shared a purchase code get new codes.** Before, several products of one purchase shared one `LOT-` code. A unique lot code is needed now, so the oldest keeps its code and the others get new ones.

**Delete is blocked for a product with history or purchase lots.** A purchase made lot can only be corrected with Adjust Stock, same as before.

## Notes

- Some test lots in the dev database have product codes like `TG12141011`, which look like generated purchase codes. They give odd stock codes such as `TG0-10-2`. This comes from that test data, not from the code format.
- Two approvals creating the same new product code at the same moment could clash on the unique index. One of them would fail and can be retried. No locking was added for this.
- The migrations only take effect after `npm run migrate` and a backend restart.

## Next steps

- Let the user edit the product code, with a uniqueness check and the existing edit request flow for non admins.
- Rename the internal `LotGroup` names (`findLotGroupById`, `inventory_lot_group`) to stock product names in a separate PR.

## Test plan

**Automated tests (no test files for this area, type check only):**
- [x] Backend `tsc --noEmit`: passes with no errors.
- [x] Backend `tsc --noEmit --noUnusedLocals --noUnusedParameters`: no unused variables or imports in the inventory files.

**Manual checks (local):**
- [x] Migration 085 in a transaction that was rolled back, on the dev data: all lots grouped, lot codes unique, product codes unique.
- [x] Migration 086 in a transaction that was rolled back: every product got a readable code and none clashed.
- [x] The list, view, lot history, summary, search and moving a lot between products, run against the migrated data in rolled back transactions.
- [ ] Approving a real purchase with two products, `updateLot` and `deleteLot` through the API, and the work order approve flow were not run end to end.
