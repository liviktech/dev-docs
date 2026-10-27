# 2026-10-27 - quantity-selector-and-cart-fix (frontend)

## Summary

Before this change a customer could not choose how many to buy on the product page, because the code always sent a quantity of 1. The mobile bottom bar had the same problem. In the cart, the quantity dropdown only went from 1 to 10, showed the number 1 twice, and used a made up limit instead of real stock.

This PR adds one reusable quantity stepper (`-`, a number box and `+`) and uses it on the product page, the mobile bar and the cart. The limit now comes from real stock when the variant tracks stock, and from the existing server limit of 100 otherwise. The type check passes and lint has no errors. I checked the stepper box live on the product page. **Adding to the cart and the cart page were not confirmed**, see "Notes" and the test plan.

**Quick add on product cards and pack sizes are not included on purpose** - see "Next steps" below. This was a choice, not a miss.

## What changed

| File | Change |
|---|---|
| `src/modules/common/components/quantity-stepper/index.tsx` (new) | The stepper. Starts from the value it is given, never goes below 1 or above the max. The box is text so the customer can clear it while typing. On blur or Enter, a letter, empty value or 0 falls back to 1, and a number above the max drops to the max. |
| `src/lib/util/inventory.ts` | Added `MAX_LINE_ITEM_QUANTITY` (100) and `getMaxQuantity(variant)`. `isVariantInStock` is untouched. |
| `src/lib/util/server-cart.ts` | The constant is now re-exported from `inventory.ts` instead of being defined here. Same value, same import path for the two routes that use it. |
| `src/modules/products/components/product-actions/index.tsx` | New `quantity` state. It goes into `addToCart` and into the hidden `quantity` field. It resets to 1 when the variant changes. Shows the stepper above the Add to cart button when the variant is in stock. |
| `src/modules/products/components/product-actions/mobile-actions.tsx` | Takes `quantity`, `setQuantity` and `maxQuantity` and shows the same stepper in the bottom bar. |
| `src/modules/cart/components/item/index.tsx` | Replaced the 1 to 10 dropdown (and the duplicate `<option value={1}>`) with the stepper. Removed the made up `maxQtyFromInventory = 10`. |
| `src/modules/cart/components/cart-item-select/index.tsx` | No real change. It is no longer used by anything, but the file was left in place. |

## Why

Vegetables are bought in bulk, so a shop that adds one piece per click loses orders. The duplicate "1" in the cart was a plain bug. Using real stock stops customers from asking for more than we have. The stepper itself was the easy part. The real work was getting one limit that works in the product page, the mobile bar and the cart.

## How

**One stepper, three places**: the same component is used everywhere, so the rules (minimum 1, clamp to max, bad input falls back) are the same on every screen.

**One limit**: `getMaxQuantity` returns the variant stock capped at 100 when the variant tracks stock. If the variant does not track stock, allows backorder, or has no stock number, it returns 100. The server cap of 100 per line is unchanged, so the customer should not hit a server error from the stepper.

**No JavaScript still works**: the hidden `quantity` input now carries the chosen number, so the plain form post to `/cart/add` sends it. That route already caps at 100.

**Cart max never below what is in the cart**: the cart passes `Math.max(maxQuantity, item.quantity)`, so a line that is already above the limit does not show a broken box.

## Decisions made

**Cart items fall back to 100 when stock is unknown.** My first version returned 1 when `inventory_quantity` was missing, which would have locked cart lines at 1 because cart line items may not carry a stock number. I changed it to fall back to 100. The server still has the final say on stock. I did not add stock fields to the cart query, to keep this PR small.

**The stepper is a text box, not a number input.** A number input allows `e`, `-` and scroll changes. A text box with digits only and a clamp on blur is simpler to keep safe.

**No `setState` inside effects for my own code.** Lint warned about two of my effects. In the stepper, the typed text is now `null` when not editing and the box shows the real value, so no effect is needed to sync it. In the product page, the reset to 1 now happens during render when the variant id changes, instead of in an effect.

**The shared constant moved to `inventory.ts`.** `server-cart.ts` is `server-only`, so client components cannot import from it. Moving the constant and re-exporting it keeps one value for both sides.

## Notes

- Stock numbers come from the product list call, which is cached for 60 seconds, so the limit can be slightly old.
- No unit tests were added. The storefront has no test setup for these components that I looked at.

## Next steps

- Quick add on product cards (`ADD` turning into this stepper) and pack sizes, in the `catalog-quick-add-and-pack-sizes` plan.
- Delete `cart-item-select` if nothing else needs it later.

## Test plan

**Automated tests (type check passing, no unit tests added):**
- [x] `npx tsc --noEmit`: passes with no errors, run after the last edit.
- [x] `npx eslint` on the changed files and the new stepper folder: 0 errors, 2 warnings, both already there before this change (see Notes).

**Manual checks (local, `/dk/products/green-brinjal`):**
- [x] Press `-` at 1: the box stayed at 1.
- [x] Press `+` four times: the box showed 5, and the hidden `quantity` input also showed 5.
- [x] Type 500 and leave the box: it dropped to 95 (the max).
- [x] Type letters and leave the box: it fell back to 1.
- [ ] Add to cart, then cart shows the same number: the cart showed 0 items after my try, cause not found.
- [ ] Cart: change quantity with the stepper, the line total updates, and 1 is not listed twice. Not run, because the cart was empty.
- [ ] Mobile width: the bottom bar adds the chosen quantity. Not run.
- [ ] JavaScript off: the form posts the chosen quantity. Not run.
- [ ] Switch variant on a product with more than one variant: quantity resets to 1. Not run, and the current catalog has one variant per product, so this needs the pack size work first.
