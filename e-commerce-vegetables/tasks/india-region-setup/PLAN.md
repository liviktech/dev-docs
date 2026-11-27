# Plan: Make the store an India store (region, country, currency)

Roadmap: Phase 1, item 2. Priority P0.

## What is wrong today

- The only region is called "Europe". Its countries are `gb, de, dk, se, fr, es, it`. This comes from the starter seed in `src/migration-scripts/initial-data-seed.ts`.
- Its currency was changed to `inr` earlier, so prices show in rupees, but everything else still says Europe.
- At checkout the country dropdown (`src/modules/checkout/components/country-select`) is built from the region's countries. An Indian customer cannot pick India, so they cannot enter an Indian address.
- The storefront URL is `/dk/...`. The default is `NEXT_PUBLIC_DEFAULT_REGION=dk` in the frontend `.env.local`.
- The tax region, shipping options and shipping zone were created for those European countries.
- The search index only keeps prices in `eur` and `usd` (`PRICE_CURRENCIES` in `src/search/helpers/pricing.ts` on the backend and `SEARCH_PRICE_CURRENCIES` in `src/lib/search-client.ts` on the frontend). Rupee prices are not indexed, so the store page price filter and prices fall back to euro numbers.

## What we are changing

1. Backend: write one repeatable script in `src/scripts/` (run with `medusa exec`) that does the following. It checks what already exists first, so it is safe to run twice.
   - Renames the region to `India`, sets the countries to only `in`, keeps currency `inr`.
   - Creates the tax region for `in` (and removes the European ones if nothing uses them).
   - Moves the shipping zone and the shipping options to `in`, and keeps their INR prices.
   - Keeps the region store policy metadata.
2. Backend: add `inr` to `PRICE_CURRENCIES` in `src/search/helpers/pricing.ts` and re-index the products.
3. Frontend: change `NEXT_PUBLIC_DEFAULT_REGION` to `in` in `.env.local` and `.env.template` if there is one.
4. Frontend: add `inr` to `SEARCH_PRICE_CURRENCIES` in `src/lib/search-client.ts`, and keep the two lists the same.
5. Frontend: check the address form. India needs a state (province) and a 6 digit PIN code, so make province required and check the PIN code format in the address form.
6. The site URL becomes `/in/...`. Add a redirect from the old `/dk` path to `/in` so saved links do not break.

## Why we are changing it

The store sells in India, so the customer must be able to enter an Indian address. Without this nobody can finish checkout in a real way. It is also the base for the payment and delivery plans, so it goes first.

## Files we expect to touch

Backend:
- New script in `src/scripts/` (for example `setup-india-region.ts`)
- `src/search/helpers/pricing.ts`

Frontend:
- `.env.local` (local only, not committed)
- `src/lib/search-client.ts`
- `src/modules/checkout/components/shipping-address/index.tsx` and `billing_address` (province and PIN code rules)
- `next.config.js` or middleware for the `/dk` to `/in` redirect

## Risks and things to watch

- Existing carts and the one test order are tied to the old region. Old carts may stop working, which is fine on a dev database. Do not run this on production data without a backup.
- Deleting the European tax regions can fail if something still points to them. Remove them only if the check says they are unused.
- The frontend caches the region list in memory (`regionMap` in `src/lib/data/regions.ts`). Restart the dev server after the change.
- The earlier session switched this region to INR and changed the shipping prices. The script must not undo those, and the old USD and EUR shipping prices are already gone.

## Out of scope

- Real GST or tax rates. We keep tax at the current setting and decide on GST separately.
- Adding other countries or currencies.

## How to test

- Open `/in`. The store loads with rupee prices.
- Checkout address step: the country is India only, state and PIN code are asked and checked.
- Delivery step still shows the two shipping options with rupee prices.
- Store page price filter shows rupee ranges.
- Open `/dk`. It redirects to `/in`.
