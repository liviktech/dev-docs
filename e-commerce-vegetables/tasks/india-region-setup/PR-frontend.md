# 2026-10-27 - india-region-setup (frontend)

## Summary

The storefront now defaults to the India region (`/in`), old country paths redirect to it, the address form asks for a State and a 6 digit PIN code, and rupee prices are known to the search client. The type check passes and lint has no errors. **The `/in` product, store and category pages returned 404 on the running dev server, so the storefront side is not confirmed.** The most likely cause is the dev server's cached region list, which needs a restart. See "Notes".

## What changed

| File | Change |
|---|---|
| `.env.local` (local only, not committed) | `NEXT_PUBLIC_DEFAULT_REGION` is now `in`. |
| `src/middleware.ts` | The fallback default is now `in`. A path that starts with an old country (`dk, gb, de, se, fr, es, it`) is redirected with a 308 to the current country, keeping the rest of the path and the query. |
| `src/lib/search-client.ts` | `SEARCH_PRICE_CURRENCIES` is now `["eur", "usd", "inr"]`, the same as the backend. |
| `src/modules/checkout/components/shipping-address/index.tsx` | "Postal code" is now "PIN code" and must be 6 digits and not start with 0 (digits keyboard, max length 6). "State / Province" is now "State" and required. |
| `src/modules/checkout/components/billing_address/index.tsx` | Same changes as the shipping address. |

## Why

The backend region is now India only, so the storefront has to use `in` as the country. The PIN code and State rules stop bad Indian addresses before they reach the order. The redirect keeps old `/dk` links working.

## How

**Redirect**: `curl -I localhost:8000/dk/store` returned `308` with `location: /in/store`.

**PIN code rule**: the browser checks `pattern="[1-9][0-9]{5}"` with a message, so no new validation code was needed.

## Decisions made

**Browser validation only.** The PIN code and State rules use HTML attributes, so they are not enforced by the server. A script that posts the form directly can skip them. A server check can be added later if needed.

**No `.env.template`.** The plan mentioned one, but this repo has none, so only `.env.local` changed.

## Notes

- `/in` returns 200, but `/in/store`, `/in/categories` and `/in/products/green-brinjal` returned 404 right after the backend change. I deleted `.next/dev/cache/fetch-cache` and cleared the cache cookie, and it stayed 404. The dev server was not restarted. The plan says the region list is cached in memory (`regionMap` in `src/lib/data/regions.ts`), so a restart is the likely fix, but this is not confirmed.
- `/dk/...` now redirects only because the old region is gone from the region list. It was not checked on a restarted server.
- Lint on the changed files: 0 errors and 2 warnings, both from an existing effect in `shipping-address/index.tsx` that I did not touch.
- No automated tests were added.

## Next steps

- Restart the storefront dev server, then run the manual checks below.

## Test plan

**Automated tests:**
- [x] `npx tsc --noEmit`: passes.
- [x] `npx eslint` on the changed files: 0 errors, 2 old warnings.

**Manual checks:**
- [x] `/dk/store` redirects to `/in/store` with a 308.
- [ ] `/in` loads with rupee prices: `/in` returns 200 but the prices were not checked.
- [ ] Product page, store page and categories load on `/in`: 404 before a restart.
- [ ] Checkout address step: country is India only, State is required, PIN code must be 6 digits: not run.
- [ ] Store page price filter shows rupee ranges: not run.
