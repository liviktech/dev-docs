# 2026-10-24 - stock-product-grouping (frontend)

## Summary

The Stock page showed one row per purchase lot. It now shows one row per product (product, type, colour, thickness). The first column is the product code and opens the view form. The view form shows the product, all its lots and where each lot came from, and the second tab shows the lot history for each lot with the same columns as before. The Glass Size column is removed from the table.

**The Available and Received sheet count columns are still there on purpose.** I took "Glass sheets" to mean the Glass Size column. This was a reading of the request, so please confirm it.

## What changed

| File | Change |
|---|---|
| `src/features/inventory/types.ts` | `lotCode` and `receivedDate` on the lot items, and `lotCode`, `purchaseNo`, `receivedDate`, `supplierName` on the history items. Comments say "stock product". |
| `src/features/inventory/stock-tab.tsx` | First column is Product Code and still opens the view form. New Type, Thickness and Lots columns. The Purchase Number column shows up to 3 purchases, then "+N more". The Glass Size column and the stacked cells are removed. KPI cards read Active Products, Products with Stock Remaining and Products Near Depletion. Delete dialog says "Delete Stock Product". |
| `src/features/inventory/components/lot-detail-dialog.tsx` | Title is "Stock Product {code}". Tab 1 "Product Details" has a details card (code, product, type, colour, thickness, lot count, sheet totals, available SQM), a Lots table (lot code, purchase no, purchased from and who created it, created date, sheet size, received, available, used, totals) and a Cutoffs table by lot code. Tab 2 "Lot History" has one table per lot with the same columns as before. A lot heading is the lot code, purchase number and date. |
| `src/features/inventory/components/lot-group-edit-dialog.tsx` | Title is "Edit Stock Product - {code}", a Lot Code column is added, and the note says a changed type, colour or thickness moves the lot to the matching product. |
| `src/features/inventory/edit-requests.ts` | Comment only, no real change. |

## Why

Stock is bought and used by product, so one row per product is easier to read and matches how glass is picked for a work order. Each lot is still visible with its purchase and supplier, so nothing about where stock came from is lost.

## How

**Product row**: the row is the product, with the lot count and the summed quantities from the backend. Lot codes are plain text inside the view and edit forms, not badges.

**Work orders**: the Select Stock screens and the work order stock section are not changed. They read per lot codes, which are now unique.

## Decisions made

**Glass Size removed, sheet counts kept.** A product has many sheet sizes, so a size column no longer fits. The sheet counts are still true per product.

**The lot heading in Lot History has no "Lot" word before the code.** It is the lot code, then the purchase number, then the date.

## Notes

- The mobile (PWA) stock list reads the raw lots, so it now shows per lot codes. It was not changed.
- `lot-edit-requests-dialog.tsx` and `lot-edit-reason-dialog.tsx` still say "lot" in their wording.
- The product codes shown come from the backend migrations 085 and 086, so they only look right after `npm run migrate` and a backend restart.

## Test plan

**Automated tests (no test files for this area, type check only):**
- [x] Frontend `tsc --noEmit -p tsconfig.app.json`: passes with no errors.
- [x] Same check with `--noUnusedLocals --noUnusedParameters`: no unused variables or imports in the inventory files.

**Manual checks (local):**
- [x] Screenshots shared by the reviewer showed the product list with product codes and the Lot History tab. The lot heading text was changed after that and not looked at again.
- [ ] Opening the Product Details tab, the edit form, and deleting a product were not checked in a browser by me.
- [ ] Approving a multi product purchase and checking the lots land in the right products was not run.
