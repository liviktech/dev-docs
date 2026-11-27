# 2026-10-27 - india-region-setup (backend)

## Summary

The only region was "Europe" with seven European countries, so an Indian customer could not pick India at checkout. This PR adds one script that turns the region into "India" (country `in`, currency `inr`), creates the Indian tax region, removes the European ones, and moves the shipping zone to India. It also adds `inr` to the search index currencies. The script was run on the local dev database and a second run changed nothing. A re-index of the products has not been checked yet, see "Notes".

## What changed

| File | Change |
|---|---|
| `src/scripts/setup-india-region.ts` (new) | Run with `npx medusa exec ./src/scripts/setup-india-region.ts`. Renames the region to India with only `in` and `inr`. Creates the `in` tax region and deletes the other tax regions that can be deleted. Sets the shipping zone to cover only `in`. |
| `src/search/helpers/pricing.ts` | `PRICE_CURRENCIES` is now `["eur", "usd", "inr"]`. |

## Why

The store sells in India. The checkout country list comes from the region, so the region has to be India for an Indian address to be entered. The search index only held `eur` and `usd`, so rupee prices were missing from the store page price filter.

## How

**Safe to run twice**: the script reads the region, tax regions and service zones first and skips anything already correct. The second run printed "Region is already India. Skipping." and changed nothing.

**Stops if there is more than one region**: it throws instead of guessing which one to change.

**Shipping prices are not touched**: only the zone countries change, so the INR prices from the earlier session stay.

**Old tax regions**: each delete is in a try and catch. If something still uses a tax region it is kept and a warning is printed. On the dev database all seven were removed.

## Decisions made

**One region renamed, not a new region created.** Renaming keeps the region id, so the shipping options, prices and metadata (store policies) stay linked.

**Tax rates are unchanged.** The new `in` tax region uses the same system provider as before. Real GST is out of scope.

## Notes

- Existing carts were tied to the old region and may stop working. This was run on the dev database only.
- The search module reindexes on the next boot after the currency list changes. The backend was not restarted, so the re-index is not confirmed.
- `npx tsc --noEmit` passes.
- No automated tests were added.

## Next steps

- Restart the backend and check the store page price filter shows rupee ranges.

## Test plan

**Automated tests:**
- [x] `npx tsc --noEmit`: passes.

**Manual checks:**
- [x] Run the script: region became India (`in`, `inr`), `in` tax region created, 7 European tax regions removed, shipping zone now `in` only.
- [x] Run it a second time: it skipped and changed nothing.
- [x] `GET /store/regions` returns one region named India with currency `inr`.
- [ ] Backend restart re-indexes products with `inr` prices: not run.
- [ ] Delivery step shows both shipping options with rupee prices: not run, needs the storefront restart first.
