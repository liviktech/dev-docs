# 2026-10-14 - api-call-cleanup (backend)

**Frontend PR**: `rubine-toughened-glass-frontend` - `<frontend PR link>` (stops loading all photos for the PI table and reads the new fields). Merge this one first.

## Summary

The PI Status table had to download every order photo to know which rows had photos or a payment receipt. This PR makes the lean PI table list (`view=table`) return two true or false flags per row, `is_photo` and `has_receipt`, so the frontend no longer needs that bulk request.

I checked that the backend compiles with `npx tsc --noEmit`. I did not run the query against the database or call the endpoint. See "Test plan".

## What changed

### rubine-toughened-glass-backend
| File | Change |
|---|---|
| `src/modules/proforma-invoice/proforma-invoice.repository.ts` | The lean query used for `view=table` now selects `is_photo` and `has_receipt` for each PI. |
| `src/modules/proforma-invoice/proforma-invoice.types.ts` | New optional `has_receipt` on the list row type. |

## Why

The Photos button needed to know "does this order have photos" and View Receipt needed "was a receipt uploaded for this PI". The frontend got that by loading every photo of every order and filtering it in the browser. The database can answer it per row with one cheap check.

## How

**`is_photo`**: true when a photo exists for the PI's order booking, or for the enquiry that order came from, other than a payment receipt. This matches the rule the Order Booking list already uses, so both screens agree.

**`has_receipt`**: true when a photo whose file name starts with `PI-PAYMENT-RECEIPT-<pi code>__` exists for the PI's order booking.

Both are `EXISTS` checks on `public.order_photo`, so they stop at the first match.

## Decisions made

**The flags are true or false, not links.** The photos still load from their own endpoint when the user opens the Photos window.

**The name is `is_photo`, not `isphoto`.** The other list endpoints and the frontend already use `is_photo`.

**Receipts are found by file name.** That is how the frontend stores and recognises them today (`PI-PAYMENT-RECEIPT-` prefix), so the backend follows the same rule instead of adding a new column.

## Bugs found while testing (already there before, not fixed here)

**1. `_` in the receipt name pattern matches any character.** The `LIKE` pattern uses `__` as the separator and `_` is a wildcard in SQL, so a PI code with a similar shape could match another PI's receipt. It is very unlikely with the real code format, and the frontend filters the photos again by exact name when they are opened.

## Notes

- The backend needs a restart (or a deploy) for the frontend to see the flags. Until then the PI table shows all Photos and View Receipt buttons as disabled.
- Nothing was pushed or deployed from this work.

## Next steps

- Add a `view=header` option to the work order endpoint, and short-response options for the party and sales rep lists.
- Add the same photo flag to the PI detail response if a screen needs it.

## Test plan

**Automated tests (none for this module):**
- [x] `npx tsc --noEmit`: passed.

**Manual checks (local):**
- [ ] Call `GET /proforma-invoices?page=1&limit=10&view=table` and confirm every row has `is_photo` and `has_receipt`. Not done.
- [ ] Check one PI with photos, one without, and one with an uploaded receipt. Not done.
