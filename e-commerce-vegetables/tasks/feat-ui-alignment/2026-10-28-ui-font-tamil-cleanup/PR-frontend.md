# 2026-10-28 - ui-font-tamil-cleanup (frontend)

## Summary

The storefront used the starter's default fonts, a mix of old text size classes and uneven weights, and the brand was spelled "Vegein". This PR switches the whole store to the Plus Jakarta Sans font, tidies headings, prices, filters, the cart, checkout, footer and navigation so sizes and weights are consistent, renames the brand to "VegIn", and stops the category list being cached in development. No Tamil text was found anywhere in the frontend source. **None of this was checked in a browser in this session.**

**Tamil words shown on product pages are not changed here on purpose** - see "Decisions made" below. This was a choice, not a miss.

## What changed

| File | Change |
|---|---|
| `src/app/layout.tsx` | Loads Plus Jakarta Sans (weights 400 to 700) with `next/font/google`, adds the font variable to `<html>`, adds Google Fonts preconnect and stylesheet links, and puts `font-sans antialiased text-ui-fg-base` on `<body>`. |
| `tailwind.config.js` | The `sans` font list now starts with the Plus Jakarta Sans variable and name, then Inter and the old list. |
| `src/styles/globals.css` | `body` and every `txt-*` class use Plus Jakarta Sans. The 10px size becomes 11px. The `text-xl`, `text-2xl` and `text-3xl` regular and semi helper classes use smaller sizes and line heights (24px to 20px, 30px to 24px, 32px to 30px). |
| `src/lib/util/brand.ts` | Brand name and description change from "Vegein" to "VegIn". |
| `public/logo.svg` | Label and text change to "VegIn". The font in the logo is now Plus Jakarta Sans. |
| `src/lib/data/categories.ts` | `listCategories` and `getCategoryByHandle` use `no-store` in development and `force-cache` otherwise, so category changes show straight away while developing. |
| `src/modules/common/components/ui/index.tsx` | `Text` only adds the default `txt-medium` class when the caller has not given a size. `Heading` only adds its default weight and size when the caller has not given one. This is what lets the pages below set their own sizes. |
| `src/modules/layout/templates/footer/index.tsx` | White background, smaller top and bottom spacing, bold uppercase column titles, semibold links, a line above the copyright. The category children list is built only when there are children. The `Text` component is replaced with a plain `<p>`. |
| `src/modules/layout/templates/nav/index.tsx`, `src/modules/layout/components/side-menu/index.tsx`, `src/modules/layout/components/cart-dropdown/index.tsx` | Consistent sizes and weights for links, menu items and the cart dropdown (shadow, border lines, a medium sized button). The cart dropdown item title tag changes from `h3` to `h4`. The side menu copyright becomes a plain `<p>`. |
| `src/modules/home/components/hero/index.tsx` | Heading is bold with tight spacing, the sub heading is medium weight, the Shop now button text is semibold. |
| `src/modules/products/components/product-preview/index.tsx`, `product-preview/price.tsx`, `src/modules/store/components/store-hits/index.tsx`, `store-hits/price.tsx` | Product cards: semibold dark title that turns green on hover, bold price, small struck through original price, sale price in green. Old commented out code removed from `product-preview/index.tsx`. Search hits now pass the title as the image `alt`. |
| `src/modules/products/components/product-price/index.tsx`, `src/modules/products/templates/product-info/index.tsx`, `src/modules/products/components/product-tabs/accordion.tsx` | Product page: bigger bold price with a green percentage badge for sales, bold title, small uppercase collection link, relaxed description text, medium accordion titles. |
| `src/modules/store/components/store-refinements/` (`current-refinements.tsx`, `on-sale-toggle.tsx`, `option-refinements.tsx`, `price-range.tsx`, `refinement-group.tsx`), `src/modules/common/components/filter-radio-group/index.tsx`, `src/modules/store/components/store-search-box/index.tsx`, `src/modules/store/components/pagination/index.tsx`, `src/modules/store/templates/index.tsx` | Filter titles are small bold uppercase, options are semibold, the price slider track is thicker, the selected option chip has a light background, pagination uses rounded buttons with the current page in green, the All products title is bold. In `price-range.tsx` the `formatAmount` function is only re-indented (no real change). |
| `src/modules/cart/templates/items.tsx`, `src/modules/cart/templates/summary.tsx`, `src/modules/cart/components/item/index.tsx` | Cart, table header, summary and item title sizes and weights. The Cart heading is now an `h1`. |
| `src/modules/checkout/components/addresses/index.tsx`, `payment/index.tsx`, `review/index.tsx`, `shipping/index.tsx` | Step headings use the same `text-xl font-bold` style and are centred vertically instead of baseline aligned. The review terms text is smaller and muted. |
| `src/modules/account/components/login/index.tsx`, `register/index.tsx` | Heading, paragraph and link styles match the rest of the store. Links are semibold and turn green on hover. |

## Why

The store should look like one product. The starter mixed many size helper classes, so headings and prices looked uneven from page to page. One font, a small set of sizes and clear weights make it easier to read, and the brand name should be spelled the same everywhere.

## How

**One font**: the font is loaded once in `layout.tsx` and exposed as a CSS variable. Tailwind's `sans` list and the `txt-*` classes in `globals.css` both use it, so old components pick it up without being edited.

**Callers set their own size**: `Text` and `Heading` used to always add a default size or weight, which fought with any class a page added. They now check the class text with a regular expression and only add the default when no size (or weight) is present. This is a guess based on class names, so an unusual class could be missed.

**Same on purpose**: product data, prices, routes and test ids (`data-testid`) are all unchanged. Only classes, a few small markup tags and the cache setting changed.

## Decisions made

**No Tamil text was found in the frontend, so nothing was removed there.** A search of the whole frontend source for Tamil script found no matches. The Tamil seen on product pages (for example "Green Brinjal (பச்சை கத்திரிக்காய்)") comes from product text stored in the backend database, so it has to be cleaned there. See the backend PR.

**Development no longer caches categories.** The `no-store` switch only applies when `NODE_ENV` is `development`. Production keeps `force-cache`.

**Google Fonts is loaded twice.** The `next/font/google` import and the manual `<link>` tags both load the font. The manual links were kept as a fallback. This is slightly wasteful.

## Notes

- `git diff` shows 37 changed files. Git also prints "LF will be replaced by CRLF" warnings for them, which is only line endings. With `--ignore-cr-at-eol` the diff is 250 added and 204 removed lines.
- `src/modules/layout/templates/footer/index.tsx` no longer imports `Text`. `clx` is still used.
- The `Text` and `Heading` change affects every page that uses them, not just the pages edited here.

## Next steps

- Look at every page in a browser, on a phone and a desktop width, because the whole look changed and nothing was viewed.
- Remove either the `next/font` import or the manual Google Fonts links in `layout.tsx`.
- Clean the Tamil text from the database (see the backend PR) and check the product pages.

## Test plan

**Automated tests:**
- [ ] `npx tsc --noEmit`: not run for this PR.
- [ ] Lint and build: not run for this PR.

**Manual checks (local):**
- [ ] Home, store, product, cart, checkout, login and register pages use Plus Jakarta Sans: not checked in a browser.
- [ ] Sale price badge, pagination, price slider and the filter chips look right: not checked in a browser.
- [ ] The logo and tab text show "VegIn": not checked in a browser.
- [x] A search of the frontend source for Tamil script found no matches (a code search, not a browser check).
