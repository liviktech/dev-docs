# Plan: Vegetable categories and home page collections

Roadmap: Phase 1, item 4. Priority P0.

## What is wrong today

- The database has four leftover clothing categories: Shirts, Sweatshirts, Pants, Merch. They come from the Medusa starter.
- None of the vegetables belong to any category. A category page such as `/categories/shirts` says "0 products".
- There are no collections at all. The home page (`src/app/[countryCode]/(main)/page.tsx`) calls `listCollections` and renders one product rail per collection (`FeaturedProducts`). With zero collections the area under the hero banner is blank.
- The backend has 100 vegetable products now (the report said 50). Each has one variant and a code that starts with `VEG-`.
- The old demo seed (`src/scripts/seed-demo-products.ts`) still creates clothing collections such as "Summer Essentials". It must not be run again.

## What we are changing

1. Write one repeatable backend script in `src/scripts/` (for example `organize-vegetable-catalog.ts`). It checks what exists before creating, so a second run changes nothing.
2. The script removes the four clothing categories (only if no product uses them).
3. The script creates the vegetable categories from the report:
   - Daily Essentials (tomato, onion, potato)
   - Gourds and Squashes
   - Green and Leafy Vegetables
   - Country Vegetables (with the Tamil name in the description)
   - Roots and Tubers
   - Anything that does not match goes into a "More Vegetables" category so no product is left without one.
4. The script puts each of the 100 products into the right category by matching words in the title (for example "Gourd" goes to Gourds). It prints a list of products that matched nothing, so a person can check them.
5. The script creates collections: "Today's Fresh Harvest", "Daily Essentials", "Leafy Greens", "Best Sellers". It fills them from the categories (a person can later change this in the admin).
6. The script re-sends the products to the search index so filters show the new categories.
7. Frontend: show a row of category cards on the home page under the hero, so customers can browse fast. Use the existing category data helper in `src/lib/data/categories.ts`.
8. Frontend: make the home page handle "no collections" without going blank (show the category cards at least).

## Why we are changing it

A shop where the home page is empty and the menu shows shirts looks broken. Categories and collections are also how customers find vegetables, and the other plans (branding, quick add) rely on a good home page.

## Files we expect to touch

Backend:
- New script in `src/scripts/`

Frontend:
- `src/app/[countryCode]/(main)/page.tsx`
- New home component for category cards in `src/modules/home/components/`
- `src/modules/home/components/featured-products` only if the empty state needs a change

## Risks and things to watch

- Matching by title words is a first pass. Some products will land in the wrong place, so the script prints a review list.
- Category and collection pages are cached for a while. A restart or the 60 second refresh may be needed before the change shows.
- The category parent and child shape: keep it flat (no sub categories) for now.

## Out of scope

- Seasonal banners and offers (comes with the branding plan).
- Product photos. Images are not changed here.

## How to test

- Run the script twice. The second run says nothing to do.
- Admin: no clothing categories, five or six vegetable categories, every product has one.
- Home page: category cards and at least two product rails show.
- `/categories/gourds-and-squashes` lists gourds.
- Search page filters by category.
