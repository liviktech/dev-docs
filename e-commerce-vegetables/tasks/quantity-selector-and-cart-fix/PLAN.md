# Plan: Quantity selector on product page and cart fix

Roadmap: Phase 1, items 1 and 3 (joined into one plan because both are about quantity). Priority P0.

## What is wrong today

- On the product page a customer cannot choose how many to buy. The code always sends `quantity: 1`. This is in `handleAddToCart` and in the hidden form field in `src/modules/products/components/product-actions/index.tsx`.
- The mobile bottom bar (`mobile-actions.tsx`) also has no quantity box. It calls the same add to cart function.
- In the cart (`src/modules/cart/components/item/index.tsx`) the quantity dropdown only goes from 1 to 10.
- The same dropdown shows the number 1 twice. There is an extra `<option value={1}>` after the loop.
- The max quantity is a made up number (`maxQtyFromInventory = 10`) with a TODO. It does not look at real stock.
- A customer who wants 15 kg of tomatoes has to add the product again and again.

## What we are changing

1. Add a small reusable quantity stepper component with `-`, a number box and `+`. It starts at 1 and never goes below 1.
2. Use the stepper on the product page. The chosen number goes into `addToCart` and into the hidden `quantity` field, so the no-JavaScript form also works.
3. Show the same stepper on the mobile bottom bar, or at least pass the chosen quantity to it, so mobile customers can also pick a number.
4. Set the top limit from real stock (`inventory_quantity`) when the product tracks stock. When it does not track stock, use the existing `MAX_LINE_ITEM_QUANTITY` (100) from `src/lib/util/server-cart.ts`.
5. In the cart, replace the 1 to 10 dropdown with the same stepper. Remove the duplicate `<option value="1">` by removing the dropdown.
6. Reset the quantity to 1 when the customer picks a different variant, and clamp it if the new variant has less stock.

## Why we are changing it

Vegetables are bought in bulk. A shop that only sells one piece per click loses orders. The duplicate option is a plain bug and looks careless. Using real stock stops customers from asking for more than we have.

## Files we expect to touch (frontend only)

- New: `src/modules/common/components/quantity-stepper/index.tsx`
- `src/modules/products/components/product-actions/index.tsx`
- `src/modules/products/components/product-actions/mobile-actions.tsx`
- `src/modules/cart/components/item/index.tsx`
- `src/modules/cart/components/cart-item-select` can stay. It is still used elsewhere, so check before deleting.

## Risks and things to watch

- The cart already limits to 100 per line on the server. The stepper must use the same limit so the user never sees a server error.
- Stock numbers come from the product list call, which is cached for 60 seconds. The number can be slightly old. The server still has the final say.
- Typing a letter, empty value or 0 in the number box must fall back to 1.

## Out of scope

- Quick add on product cards (see the `catalog-quick-add-and-pack-sizes` plan).
- Weight or pack choices (same plan).

## How to test

- Product page: press `+` to 5, add to cart, the cart shows 5.
- Press `-` at 1, it stays at 1.
- Type 500, it drops to the max.
- Cart: change quantity with the stepper, the line total updates, the number 1 is not listed twice anywhere.
- Mobile width: the bottom bar adds the chosen quantity.
- Run with JavaScript off: the form posts the chosen quantity.
