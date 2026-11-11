# 2026-10-28 - pagination-and-thumbnail-fit (frontend)

## Summary

The storefront previously had two separate pagination implementations that looked inconsistent: the category pages used router navigation with no previous or next arrows, while the search hits page used raw text buttons with no styling. Additionally, product thumbnail images were using `object-cover` inside a tall container, which zoomed in and cropped out parts of the vegetable photos. This PR adds a reusable pagination component with a green active badge, inactive buttons, previous and next arrow buttons (`<` and `>`), and ellipsis handling, replaces both old pagination implementations with this single component, and updates product thumbnails to use `object-contain` with padding so the full vegetable photo is visible without altering card widths. These changes were checked live against the running development server.

## What changed

| File | Change |
|---|---|
| `src/modules/common/components/pagination/get-page-items.ts` (new) | Computes the list of page numbers and ellipsis tokens to render. Shows up to 7 pages without gaps, and truncates larger page ranges with ellipses around the current page and endpoints. |
| `src/modules/common/components/pagination/index.tsx` (new) | Reusable pagination component. Renders previous and next chevron buttons, a green active page badge (`bg-brand-600`), and inactive page buttons. Supports both direct `onPageChange` callbacks (used by InstantSearch) and standard Next.js URL query parameter navigation (`?page=N`) wrapped in a `<Suspense>` boundary. |
| `src/modules/common/icons/chevron-left.tsx` (new) | SVG icon component rendering a left-pointing chevron for previous page navigation. |
| `src/modules/common/icons/chevron-right.tsx` (new) | SVG icon component rendering a right-pointing chevron for next page navigation. |
| `src/modules/products/components/thumbnail/index.tsx` | Replaces `object-cover` with `object-contain object-center p-2` and increases image quality to 80 so the whole vegetable photo fits inside the card without edge cropping or blurriness. |
| `src/modules/store/components/pagination/index.tsx` (deleted) | Removed the old store-specific pagination component because callers now use the shared component under `@modules/common/components/pagination`. |
| `src/modules/store/components/store-hits/pagination.tsx` | Refactored InstantSearch pagination to use the new shared `Pagination` component with `refine()`, giving the store search page the exact same styling and arrow buttons. |
| `src/modules/store/templates/paginated-products.tsx` | Switched import to the shared pagination component and removed redundant `totalPages > 1` wrapper check since the component hides itself on single pages. |

## Why

Pagination had inconsistent styling across the site and lacked quick navigation arrows for previous and next pages. In addition, product images were getting cut off because `object-cover` forced wide and square vegetable photos to fill a tall vertical card. Unifying pagination into a single reusable component makes page navigation consistent everywhere, and switching thumbnails to `object-contain` ensures customers can see the entire vegetable before clicking.

## How

**Shared pagination with dual navigation modes**: The `Pagination` component accepts an optional `onPageChange` prop. When provided (such as on `/store` with InstantSearch), clicking a button invokes the callback. When omitted (such as on category and collection pages), it updates the `page` query parameter using Next.js navigation.

**Suspense boundary for search parameters**: Because Next.js App Router requires `useSearchParams` to be wrapped in a suspense boundary during client navigation, the router logic is isolated into an inner `PaginationWithSearchParams` component wrapped in `<Suspense fallback={null}>`.

**Clean pagination range computation**: Logic for determining when to display page numbers versus ellipsis placeholders (`...`) is factored out into `get-page-items.ts` to keep the UI component focused and easily testable.

**Preserved card widths with full image containment**: In `thumbnail/index.tsx`, container widths, heights, and aspect ratios were kept untouched. Only the image fit style was changed from `object-cover` to `object-contain` with `p-2` padding so photos never bleed into or get clipped by the rounded card container.

## Decisions made

**Used `object-contain` instead of resizing card aspect ratios.** Changing the container aspect ratio across cards would alter existing grid layout heights and require responsive redesigns. Using `object-contain` with internal padding solved the image cropping immediately while preserving card dimensions and alignment.

**Placed pagination under `common/components` instead of `store/components`.** Pagination is a generic UI control that may be reused in other sections like order histories, account tables, or reviews in the future, matching components like `quantity-stepper` and `filter-radio-group`.

**Disabled navigation arrows on boundaries rather than hiding them.** The previous arrow `<` is disabled on page 1 and the next arrow `>` is disabled on the final page, retaining the fixed width and layout of the pagination bar without shifting elements.

## Notes

- Both page 1 and page 2 of `/in/categories/country-vegetables` were checked against the running Next.js server (`http://localhost:8000/in/categories/country-vegetables`) to verify rendered markup, disabled arrow states, and active badge styling.
- The `src/modules/store/components/pagination` directory was removed because all imports were pointed directly to `@modules/common/components/pagination`.

## Test plan

**Automated tests:**
- [x] Node.js live HTML validation: fetched `http://127.0.0.1:8000/in/categories/country-vegetables` and verified `<nav aria-label="Pagination">`, `<button aria-label="Previous page" disabled>`, active page 1 button (`bg-brand-600`), and next page arrow.
- [x] Node.js live HTML validation for page 2: fetched `http://127.0.0.1:8000/in/categories/country-vegetables?page=2` and verified previous arrow is enabled, page 2 is active, and next arrow is disabled.
- [ ] `npm run build`: not run in this session to keep the running local development server active.

**Manual checks (local):**
- [x] Verified category pagination renders with `<` and `>` chevron buttons and green badge on active page.
- [x] Verified `/in/store` loads with status 200 and uses the new unified pagination component.
- [x] Verified vegetable thumbnails on `/store` and category pages fit completely within their cards without edge cropping.

