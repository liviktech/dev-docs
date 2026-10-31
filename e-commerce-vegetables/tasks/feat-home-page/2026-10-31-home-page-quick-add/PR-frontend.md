# 2026-10-31 - home-page-quick-add (frontend)

## Summary

The home page was a hero banner and a trust strip, the header was a plain white bar, and a shopper had to open a product page to add anything to the cart. This PR gives the storefront an Amazon style layout. The header is now a dark green bar with the logo, a "Deliver to" address, a search box with suggestions, a language dropdown, the cart and the account link. The home page shows a "Buy it again" row (signed-in customers only), a "Shop by category" grid, and one scrolling row per collection. Collection and category pages now use the same filter sidebar as the store page. Every product card has a round green "+" on the picture that turns into a "− 1 +" pill, so items can be added and removed without leaving the page. The separate Add button under the cards is gone from all pages.

Checked live in the browser (English): the header, the home page, the store page, adding, removing and changing quantities from the cards, the header cart count, and that quantities carry over from the home page to the store page. Type check passes and lint shows no errors on the quick add, cart quantity, hook and search hit files.

**Pack-size dropdown on the cards, the "Only N available" message and the error message were not checked with real data.** All 100 products in the local store have one variant and plenty of stock, so none of these could show up. **Tamil and Hindi wording for the new text was written by me and has not been reviewed by a native speaker.**

## What changed

Three files (`paginated-products.tsx`, `related-products/index.tsx` and the two skeleton templates) only show line ending changes in `git status`. They have no real change, so skip them.

| File | Change |
|---|---|
| `src/lib/context/cart-quantities.tsx` (new) | `CartQuantitiesProvider` and `useCartQuantities()`. Keeps the quantity of each variant in the cart, shared by every card on the page. Changes show instantly and are saved to the cart in the background. |
| `src/lib/util/quick-add.ts` (new) | `toQuickVariants(product)` turns a product into a small list of variants (id, title, price, in stock, max quantity), cheapest first. Variants with no price are dropped. |
| `src/lib/data/quick-add.ts` (new) | Server action `listQuickAddVariants(productIds, countryCode)`. Loads full products by id (up to 48) for cards that only have search data. |
| `src/lib/hooks/use-quick-variants.ts` (new) | Client hook for the store, collection and category lists. Fetches the variants for the hits on screen in one call and caches them for 60 seconds. |
| `src/modules/products/components/quick-add/index.tsx` (new) | The control on the card. Round green "+" (`brand-700`, hover `brand-800`, drawn as an SVG so it sits exactly in the middle), a white "− 1 +" pill once the item is in the cart, an "Out of stock" pill, and a pack-size dropdown when a product has more than one variant. |
| `src/modules/products/components/product-preview/index.tsx` | The card is now picture (link) with the control on top, then a second link with the title and price. Used by the home rows, search and related products. |
| `src/modules/store/components/store-hits/index.tsx` | Same card layout for the store, collection and category lists, with the control fed by `useQuickVariants`. |
| `src/modules/home/components/buy-again/index.tsx` (new) | "Buy it again" row. Looks at the customer's last 20 orders, ranks products by how often and how recently they were bought, and shows up to 12 that are still in stock. Hidden for signed-out customers and customers with no orders. |
| `src/modules/home/components/category-grid/index.tsx` (new) | "Shop by category" grid of top-level categories, each pictured with the first product thumbnail in it, plus an "All products" link. |
| `src/modules/home/components/scroll-row/index.tsx` (new) | One row that scrolls sideways with previous and next buttons. 2 cards on phones, 4 from 1024px up. |
| `src/modules/home/components/featured-products/product-rail/index.tsx` | Collection rows now use `ScrollRow`, a smaller heading and tighter spacing. Fields now include `+variants.inventory_quantity` so the card knows stock. A collection with no products returns nothing. |
| `src/modules/home/components/hero/index.tsx` (deleted) | The hero banner is replaced by the new home sections. |
| `src/app/[countryCode]/(main)/page.tsx` | Home page now renders Buy it again, the category grid, the trust strip and a row per collection. |
| `src/app/[countryCode]/(main)/layout.tsx` | Wraps the page in `CartQuantitiesProvider`. |
| `src/modules/layout/templates/nav/index.tsx` | New dark green header. The VegIn name is drawn next to a small leaf logo. The side menu, old search and inline language switcher are gone from here. Order of items changes on phones so search drops to its own row. |
| `src/modules/layout/components/delivery-address/index.tsx` (new) | "Deliver to" block. Shows the signed-in customer's default shipping address (city and postal code), or a link to sign in or add an address. |
| `src/modules/layout/components/language-dropdown/index.tsx` (new) | Header language picker. Shows the short code and lists English, Tamil and Hindi. Uses the same `setLanguage` action as before. |
| `src/modules/layout/components/nav-search/index.tsx` (new) | Header search box with a suggestions list (up to 8). Enter goes to the store page with the query in the URL. |
| `src/modules/layout/components/search/hit.tsx` | New `compact` option for the smaller rows in the suggestions list. Also fixes a line that was joined by mistake (`sizes` and `className` were on one line). |
| `src/modules/layout/components/cart-dropdown/index.tsx` | Cart link is now a cart icon with a count badge and the word "Cart" on wider screens. |
| `src/modules/layout/templates/footer/index.tsx` | Footer colours changed to the dark green with white text. No change to the links or text. |
| `src/modules/store/templates/index.tsx` | Takes `header`, `productIds`, `hideCategoryFilter` and `testId`. When `productIds` is given the search client adds that scope to every request, so the filters work inside a collection or category. |
| `src/modules/store/components/store-query/index.tsx` (new) | Renders nothing. Mounts `useSearchBox` so `?product[query]=` still works now that the search input lives in the header. |
| `src/modules/store/components/store-search-box/index.tsx` (deleted) | The in-page search box, replaced by the header search. |
| `src/modules/store/components/store-refinements/index.tsx` | New `hideCategory` prop. |
| `src/modules/collections/templates/index.tsx`, `src/modules/categories/templates/index.tsx` | Both now render `StoreTemplate` with the product ids of the collection or category (a category includes its sub-categories). The collection page hides the Category filter. Sort, page and option filters now come from the InstantSearch URL, not server props. |
| `src/app/[countryCode]/(main)/collections/[handle]/page.tsx`, `src/app/[countryCode]/(main)/categories/[...category]/page.tsx` | No more `searchParams`. They load the region and pass its currency code. |
| `src/lib/data/categories.ts` | `getCategoryByHandle` also loads `*category_children.products`, so a category page can list products from its sub-categories. |
| `src/lib/i18n/dictionaries/en.ts`, `ta.ts`, `hi.ts` | New keys: `common.previous`, `common.next`, `nav.addAddress`, `nav.deliverTo`, `nav.deliveringTo`, `nav.signInForAddress`, `nav.updateLocation`, `home.shopByCategory`, `home.buyAgain` and eight `quickAdd.*` keys. |

## Why

Shoppers buying vegetables add many items at once, and opening a product page for each one is slow. A "+" and "−" on every card is the common fix, and the Amazon style header and home sections give a shopper somewhere to start (their earlier orders, then categories, then collections). The real work was keeping the cart in step: the cards on a page must all agree with the cart, the first add must not create two carts, and quick taps must not flood the server.

## How

**Shared cart quantities**: one provider holds the quantity per variant. It loads the cart on mount, on every page change, and when the header cart announces an update. Cards read from it, so the same item shows the same number everywhere on the page.

**Instant changes, saved later**: a tap updates the number at once. Taps on the same variant are joined and sent 350 ms after the last one. Saves run one at a time through a single queue. This is what stops the first add from creating two carts. Calls are `addToCart`, `updateLineItem` and `deleteLineItem`, and after each one the header cart count is refreshed.

**Failures**: if the cart rejects a change, the number goes back to what the cart really holds and the card shows "Couldn't update your cart. Please try again."

**Stock**: "+" stops at the variant's stock (capped at 100) and the card shows "Only N available". An out of stock variant shows an "Out of stock" pill instead of the "+".

**Two kinds of card data**: home rows, search and related products already have full products, so they pass variants straight in. The store, collection and category lists come from the search index, which only has a price range, so `useQuickVariants` fetches the variants for the visible hits in one call. If that call fails, the controls are hidden for those cards instead of showing a wrong one.

**Scoped listing**: collection and category pages reuse the store page. The product ids go into every search request as an extra `id` filter. It is done on the client, not with `<Configure>`, because choosing a filter replaces `Configure`'s own filters and would drop the scope. A collection or category with no products matches nothing instead of falling back to the whole catalogue.

**Kept the same on purpose**: the cart server actions, the stock helpers, the brand name "VegIn" (never translated), and the existing filter widgets are unchanged.

## Decisions made

**Controls sit on the picture, not under it.** The first version had them below the card. They were moved onto the image, and the old Add button on the "Buy it again" row (`add-button.tsx`, with its `home.buyAgainAdd` and `home.buyAgainAdded` text) was deleted, so there is one way to add from a card on every page.

**The "+" is an SVG, the circle is `brand-700`.** The first version used a "+" text character, which sat slightly high in the circle, and a red circle. It now uses the app green and the icon centre matches the circle centre (measured 0 and 0 offset in the browser).

**Variants are loaded after the page for search based lists.** Another option was to add variant data to the search index. That needs a backend and indexing change, so it was not done here.

**Pack-size picker only appears for products with more than one variant.** Single pack products stay as a clean circle.

**Taps build on the latest quantity, not the one on screen.** The first version computed the new number from the quantity of the last render, so several taps in the same instant (a fast double tap) all started from the same value and some were lost. `setQuantity` now also accepts a function that gets the latest wanted quantity, and the card sends "+1" or "−1" (kept between 0 and the stock limit).

## Notes

- The local store has 100 products, each with one variant, so the pack-size dropdown was never shown.
- Test items added during checking were removed again.
- Home page returned a 500 for a short time while building, with "useCartQuantities must be used inside CartQuantitiesProvider". The layout import was added before the wrapper. Fixed in the same change, nothing to do.
- Nothing is committed. The work is uncommitted changes on `feat/home-page`.

## Next steps

- Have someone who reads Tamil and Hindi review the new text (the `quickAdd.*`, `nav.*` and `home.*` keys).
- Add a product with two or more pack sizes in the local store and check the dropdown, the "N in cart" label and the stock limit message.
- If search lists feel slow on first load, consider putting variant and stock data in the search index so the extra fetch is not needed.

## Test plan

**Automated tests (no test files in this PR):**
- [x] Type check (`npx tsc --noEmit`): passes with no errors.
- [x] Lint (`npx eslint` on `quick-add`, `src/lib/context`, `use-quick-variants.ts` and `layout/components/search`): no errors. Warnings of the "refs" kind in `store-hits` were there before.

**Manual checks (local, http://localhost:8000/in, English):**
- [x] Home page loads with the new header, category grid and collection rows.
- [x] Clicked "+" on a card, then "+" again, then "−": quantity went 1, 2, 1, and the header showed the same count (for example "Cart (3)" after three adds).
- [x] Fired 4 "+" clicks in the same instant after the first add: quantity showed 5. Then 3 "−" clicks in the same instant: showed 2, and the header settled on 2 as well.
- [x] Clicked "−" at 1: the item was removed, the card went back to the "+" circle and the header showed "Cart (0)".
- [x] Added items on the home page, opened the store page: the same quantities showed there.
- [x] Store page: controls are on the pictures, none left under the cards.
- [x] Phone width (375px): tap targets are large enough.
- [x] "+" circle background is `rgb(21, 128, 61)` and the icon is centred (0 offset).
- [ ] Final screenshot of the green centred "+" - the browser pane timed out when taking it. The measurement above covers it.
- [ ] Pack-size dropdown, stock limit message and error message - not possible, every local product has one variant with plenty of stock.
- [ ] Tamil and Hindi views of the new header, home sections and card text - not checked live.
- [ ] "Buy it again" row with a real customer who has orders - not checked live.
- [ ] Delivery address block signed in with a saved address - not checked live.
