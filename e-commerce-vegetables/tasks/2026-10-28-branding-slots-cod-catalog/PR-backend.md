# 2026-10-28 - branding-slots-cod-catalog (backend)

## Summary

The backend had no delivery slot support, the India region did not offer the manual payment provider, and the catalog still had the starter's clothing categories with no vegetable categories or home page collections. This PR adds delivery slots (two store routes, a check when the order is placed, and an admin widget), a script that turns on Cash on Delivery for the region, and a script that sets up the vegetable categories and collections. The type check passes. **The routes, the hook, the widget and both scripts were not run in this session.**

**Slot cut off times, per slot order limits, closing days and a "slot is full" check are not included on purpose** - see "Decisions made" below. This was a choice, not a miss.

## What changed

| File | Change |
|---|---|
| `src/lib/delivery-slots.ts` (new) | Slot settings and helpers. Two slots (`morning` 7 to 10 AM, `evening` 5 to 8 PM), 4 days listed starting today in India time. `listSlotOptions` builds the list and `checkSlot` checks a date and slot. |
| `src/api/store/delivery-slots/route.ts` (new) | `GET /store/delivery-slots` returns the list of dates and slots. |
| `src/api/store/carts/[id]/delivery-slot/route.ts` (new) | `POST /store/carts/:id/delivery-slot` checks the date and slot, then saves `delivery_date` and `delivery_slot` in the cart metadata, keeping the other metadata. |
| `src/api/middlewares.ts` | Adds body validation (`date` and `slot` as strings) for the new POST route. The existing store search config is the same. |
| `src/workflows/hooks/validate-delivery-slot.ts` (new) | A hook on `completeCartWorkflow` that refuses the order when the cart has no valid delivery date and slot. |
| `src/admin/widgets/order-delivery-slot.tsx` (new) | Admin widget on the order page (`order.details.side.after`) that shows the chosen date and slot, or "No delivery slot was chosen." |
| `src/scripts/enable-cod.ts` (new) | Adds `pp_system_default` to the India region's payment providers. Safe to run twice. |
| `src/scripts/organize-vegetable-catalog.ts` (new) | Removes the clothing categories, creates the vegetable categories and four collections (Today's Fresh Harvest, Daily Essentials, Leafy Greens, Best Sellers), and files each product by words in its title. Safe to run twice. |

## Why

Vegetables are bought for a meal today or tomorrow, so the packing team needs the day and time. Medusa has no delivery slot, but carts and orders have a `metadata` field that moves from the cart to the order, so no new table is needed. Cash on Delivery is the payment method for launch, and Medusa's built in manual provider does the job. The catalog script replaces clicking through the admin for every product.

## How

**Slot rules in one file**: the slot names, labels and number of days live in `src/lib/delivery-slots.ts`. Dates are worked out in India time (UTC plus 5:30) on the server, not from the browser clock.

**Saving a slot**: the route reads the cart, then writes the old metadata plus the two new keys, so other metadata is not lost.

**Blocking an order**: the complete cart hook runs the same `checkSlot` check, so a cart without a slot cannot become an order even if someone skips the checkout page.

**Cash on Delivery**: `enable-cod.ts` stops with an error unless there is exactly one region, and only updates the region when the provider is missing.

**Catalog script**: categories and collections are found by handle, and a product is only updated when its category or collection differs. It refreshes the search index at the end and prints the products under each category and collection. Products that match no rule go to "More Vegetables" and are listed for review.

## Decisions made

**Placeholder slot rules.** The plan asked for cut off times, a maximum number of orders per slot and a limit on days ahead. This PR has none of the cut offs or limits, and the file says so. The 4 listed days include today even if a slot has already started. These rules need the owner's answers first.

**The order check is light.** The hook only checks that a valid slot exists. It does not check that the slot is still in the future when the order is placed.

**Collections do not overlap.** A product can only sit in one collection in Medusa, so each product goes in at most one of the four collections.

**Scripts, not migrations.** Both data changes are scripts run by hand with `npx medusa exec`, so they can be read and re-run, and nothing changes on its own at startup.

## Notes

- Run order for a fresh setup: `npx medusa exec ./src/scripts/enable-cod.ts`, then `npx medusa exec ./src/scripts/organize-vegetable-catalog.ts`. The catalog script's own notes say to check the review list it prints, because products are matched by words in the title.
- The storefront caches for about 60 seconds, so catalog changes may take a minute to show.
- `npx tsc --noEmit` passes in the backend. No automated tests were added.
- The admin widget keeps its own copy of the slot labels, so a label change has to be made in `delivery-slots.ts`, the widget and the storefront `delivery-slot.ts`.

## Next steps

- Get the cut off times, order limits and closing days from the owner, then add them to `src/lib/delivery-slots.ts` and the hook.
- Run both scripts on the real database and read the printed review list.
- Move the slot labels to one shared place so they are not repeated in three files.

## Test plan

**Automated tests:**
- [x] `npx tsc --noEmit`: passes with no errors.

**Manual checks (local):**
- [ ] `GET /store/delivery-slots` returns 4 days with 2 slots each: not run.
- [ ] `POST /store/carts/:id/delivery-slot` with a good date and slot saves the metadata, and a bad slot returns an error: not run.
- [ ] Completing a cart with no slot is refused with "Please choose a delivery date and time slot.": not run.
- [ ] The admin order page shows the delivery slot widget: not run.
- [ ] `enable-cod.ts` run twice: the first run adds the provider, the second says "Nothing to do": not run.
- [ ] `organize-vegetable-catalog.ts` run twice: the second run changes nothing: not run.
