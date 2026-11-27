# 2026-11-10 - pwa-reports-item-colours (backend)

## Summary

An item is one product + glass type + thickness with a list of colours. Before this change a blank glass type, colour or thickness on an item acted like a wildcard, so it matched any purchase line or rate row, and a purchase line sent with the wrong item was rejected. This PR makes the backend strict (a blank matches nothing), picks the item that really fits a purchase line, and adds a product filter and wider search on the Items list. I only type checked it, I did not call the endpoints live.

The mobile Reports screens are frontend only, so **no backend change was made for them on purpose**. They use the existing report endpoints as they are.

## What changed

| File | Change |
|---|---|
| `src/modules/inventory-purchase/inventory-purchase.service.ts` | `resolveItems` now finds the item that fits each purchase line (product + glass type + colour + thickness). If the item sent does not fit, it takes the product's active Purchase item that does. If none fits it throws `INVENTORY_PURCHASE_ITEM_NOT_FOUND`. The line is saved against the item that fits. |
| `src/modules/master/item/item.repository.ts` | New `findActivePurchaseItemsOfProducts`. `listItemPage` gets a `productName` filter, and the search now also matches thickness and colour name or code. The count query uses the `i` alias so the colour search works. |
| `src/modules/master/item/item.controller.ts` | `listItems` reads the `productName` query param (trimmed) and passes it on. |
| `src/modules/master/product/product.repository.ts` | `findRateMismatch` no longer treats a blank glass type or thickness on an item as a match. |

## Why

The Items page and the product Rates tab now treat an item as a full combination, so the backend checks had to agree. If they did not, the screens would show "no item" while the backend quietly accepted a blank item, or a purchase would fail on a mismatch the user could not see.

## How

**Strict match**: An item fits a line only when it has a glass type, a thickness, and a colour list that includes the line's colour, and all of them equal the line. A blank value matches nothing.

**Purchase line item**: If the line has no `itemCode`, the product is taken from `productId`. If it has one that does not fit, the product of that item is used. The sibling items of those products are loaded in one query and the first one that fits (ordered by code) is used.

**Items list**: `productName` is matched ignoring case. The search box now also matches `thickness_mm` as text and the colours of the item by name or code.

**Kept the same on purpose**: The checks for an unknown or inactive item, an item with no product, an item that is not a Purchase item, and a product that is not available for Purchase are unchanged, and so is the HSN fallback.

## Decisions made

**The sent item is replaced, not rejected.** Before, a wrong item gave `INVENTORY_PURCHASE_ITEM_MISMATCH`. Now the line is moved to the item that fits, because the purchase form lets the user change type, colour and thickness after picking an item. A hard error only happens when no item fits at all.

**Blank means no match.** I did not keep the old "blank matches anything" rule for old items. Older items with blank fields will stop matching until they are filled in on the Items page.

## Notes

- `INVENTORY_PURCHASE_ITEM_MISMATCH` is no longer thrown from this service, the new error is `INVENTORY_PURCHASE_ITEM_NOT_FOUND`. Anything that shows the old code needs to know about the new one.

## Test plan

**Automated tests (no tests added):**
- [x] `npx tsc --noEmit` (backend): ran in this session, no errors.

**Manual checks (local):**
- [ ] Create a purchase with a line whose item does not fit, and see it saved against the fitting item - not run against a live backend.
- [ ] Create a purchase for a combination with no item and see `INVENTORY_PURCHASE_ITEM_NOT_FOUND` - not run.
- [ ] `GET` items with `productName` and with a thickness or colour search - not run.
