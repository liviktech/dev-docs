# Plan: Quick add on product cards and pack size choices

Roadmap: Phase 2, items 1 and 2 (joined into one plan because a card can only quick add if it knows which pack size to add). Priority P1.

## What is wrong today

- A product card (`src/modules/products/components/product-preview/index.tsx`) is only a link. The customer must open each product to add it.
- Every product has exactly one variant and one option (the database shows 100 products, one variant each). There is no 250 g, 500 g or 1 kg choice. The pack text only lives in the description.
- Because of that, the price shown is for one fixed pack, and there is no price per kg to compare.
- The product page already knows how to show option buttons (`OptionSelect`) but there is only one option, so nothing is shown.

## What we are changing

Pack sizes (backend, data):
1. Write a repeatable backend script that adds an option called `Pack size` to each product with the values `250 g`, `500 g`, `1 kg`. A fourth value `5 kg (bulk)` is added only for products marked as bulk friendly.
2. The script creates one variant per pack size, with its own code (for example `VEG-TOMATO-250G`), its own price and its own stock.
3. Price rule: take the current price as the 1 kg price, then 500 g is 55 percent and 250 g is 30 percent, rounded to a whole rupee. The shop owner can change prices later in the admin. This is a starting point, not a final price list.
4. Stock: each new variant gets inventory at the existing stock location. Keep the existing rule of 100 units each.
5. Keep the existing variant as the 1 kg pack so old carts and the one test order stay valid.
6. Add a short "price per kg" note in the product metadata or compute it on the page.

Quick add (frontend):
7. Add a client component `QuickAdd` on the product card. It shows an `ADD` button. After one click it turns into `- 1 +` using the quantity stepper from the `quantity-selector-and-cart-fix` plan.
8. If a product has more than one variant, the button adds the default (smallest in stock) pack, and a small pack picker on the card lets the customer change it.
9. Card quantity is read from the cart, so it stays correct after a refresh. Update uses the existing `addToCart` and `updateLineItem` and `notifyCartUpdated`.
10. The card is still a link for the picture and name. The quick add area must not trigger that link (stop the click from bubbling).
11. Show the price per kg under the price on the product page and on the card.

## Why we are changing it

Grocery customers add many items fast and expect to pick a weight. This is the main thing that makes the store feel like a vegetable shop and not a generic template. It also makes the price easier to compare.

## Files we expect to touch

Backend:
- New script in `src/scripts/` (for example `add-pack-size-variants.ts`)

Frontend:
- `src/modules/products/components/product-preview/index.tsx`
- New `src/modules/products/components/quick-add/index.tsx`
- `src/modules/products/components/product-actions/index.tsx` (small, for price per kg)
- `src/lib/util/get-product-price.ts` (price per kg helper)

## Risks and things to watch

- This plan depends on the stepper component from the quantity plan. Build that first.
- Product cards are server components today, so the quick add must be a small client component inside them.
- The cart needs to be known on the card. Fetch the cart once at the page level and pass the quantities down, not once per card (100 cards would mean 100 calls).
- The search index stores one price range per product. After adding variants, re-send products to the index so the range is right.
- Wrong price maths is costly. Print the new price table from the script and have the owner approve it before the real run.
- The pack size rules are a guess. **We are not handling per-piece or per-bunch items (coriander, mint) here.** They need a different option such as `1 bunch`. List them for the owner to decide.

## Out of scope

- Real farm gate pricing and daily price changes.
- Selling by exact weight (for example 1.35 kg).

## How to test

- Admin: a product has three variants, each with a price and stock of 100.
- Product page: pack size buttons show, the price changes with the choice, price per kg shows.
- Home and store grid: `ADD` turns into a stepper, the navbar cart count follows.
- Refresh the page: the card still shows the quantity in the cart.
- Clicking the picture still opens the product page.
