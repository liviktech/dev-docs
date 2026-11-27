# 2026-10-28 - branding-slots-cod-catalog (frontend)

## Summary

The storefront still looked like the Medusa demo, had no delivery date or time at checkout, showed "Manual Payment" as the only payment option, and crashed on every request when the backend was not running. This PR gives the store its own "Vegein" branding, adds a delivery time picker to checkout (shown again in review and on the order pages), shows the manual payment provider as Cash on Delivery, and makes the middleware fall back to the default region when the backend cannot be reached. The type check passes. **None of this was checked in a browser in this session**, see "Test plan".

**A real logo, hero photo, share image and the WhatsApp, email and address details are not included on purpose** - see "Decisions made" below. This was a choice, not a miss.

## What changed

| File | Change |
|---|---|
| `src/lib/util/brand.ts` (new) | The one place for brand text: name, tagline, promise, description, phone, WhatsApp, email, address, logo path, hero image path. Also holds the three trust points and the `phoneLink` and `whatsappLink` helpers. Contact values can come from `NEXT_PUBLIC_STORE_*` env values. |
| `src/lib/util/site.ts` | `STORE_NAME` now reads `BRAND.name` instead of "Medusa Store". |
| `src/app/layout.tsx` | Adds a default title, a title template (`%s \| Vegein`), a description and the Open Graph site name. |
| `src/app/[countryCode]/(main)/page.tsx` | Home page title and description come from `BRAND`. The home page now shows the hero and the trust strip. The featured products block and the collections lookup were removed from this page. |
| `src/app/[countryCode]/(main)/products/[handle]/page.tsx`, `categories/[...category]/page.tsx`, `collections/[handle]/page.tsx` | Page titles no longer repeat the store name, because the layout template adds it. |
| `src/app/[countryCode]/(main)/account/@login/page.tsx`, `account/@dashboard/profile/page.tsx` | Descriptions no longer say "Medusa Store". |
| `src/app/[countryCode]/(checkout)/layout.tsx` | Header shows `BRAND.name`. The "Powered by Medusa" block is removed. |
| `src/modules/home/components/hero/index.tsx` | New hero: green background (or a photo when `BRAND.heroImage` is set), tagline, promise and a "Shop now" button to `/store`. The GitHub button is gone. |
| `src/modules/home/components/trust-strip/index.tsx` (new) | Three trust points under the hero. |
| `src/modules/layout/templates/nav/index.tsx` | Shows the logo image instead of the "Medusa Store" text. |
| `src/modules/layout/templates/footer/index.tsx` | Removes the GitHub, Docs, Source code links and the Medusa call to action. Adds a "Contact us" block (phone, WhatsApp, email, address, each shown only when set) and the tagline. |
| `src/modules/layout/components/side-menu/index.tsx` | Copyright line uses `BRAND.name`. |
| `src/modules/account/components/register/index.tsx` | "Become a member" wording and `BRAND.name` in the terms line. |
| `src/modules/policies/templates/index.tsx` | Placeholder notice no longer mentions the Medusa starter. |
| `public/logo.svg` (new), `src/app/icon.svg` (new) | Placeholder logo and favicon. |
| `src/app/opengraph-image.tsx` (new) | Placeholder share image: brand name and tagline on green. |
| `tailwind.config.js`, `src/styles/globals.css` | Adds a `brand` green palette and turns the Medusa UI "interactive" colours (links, focus, selected) green in light mode. |
| `src/lib/util/delivery-slot.ts` (new) | Reads the chosen date and slot from cart or order metadata and formats them for display. |
| `src/lib/data/cart.ts` | Two new server actions: `listDeliverySlots` and `setDeliverySlot`. |
| `src/modules/checkout/components/delivery-slot/index.tsx` (new) | "Delivery time" section with date chips and slot buttons. |
| `src/modules/checkout/components/shipping/index.tsx` | Shows the picker under the shipping options. "Continue to payment" stays disabled until a slot is chosen. The collapsed summary shows the chosen time. |
| `src/modules/checkout/components/review/index.tsx` | Shows the delivery time and uses `BRAND.name` in the privacy line. |
| `src/modules/order/components/shipping-details/index.tsx` | Shows the delivery time on the order page. |
| `src/lib/constants.tsx` | `pp_system_default` is now "Cash on Delivery" with a cash icon and a short description. The `paymentInfoMap` type gets an optional `description`. |
| `src/modules/checkout/components/payment-container/index.tsx` | Shows the description under the title. The development only "test payment" label is removed. |
| `src/modules/checkout/components/payment/index.tsx` | For the manual provider the card says "Pay in cash when your order arrives". |
| `src/modules/order/components/payment-details/index.tsx` | For the manual provider it says the amount is to be paid in cash on delivery. An unknown provider no longer crashes the page. |
| `src/middleware.ts` | The regions fetch is wrapped in try/catch. If the backend is down or answers with an error, it logs the problem and returns the current (possibly empty) map, so the default region `in` is used. |

## Why

The starter text made the store look unfinished, and a single source for brand text means a rename later takes one edit. Fresh vegetables are time sensitive, so the packing team needs to know the day and the time the customer wants. Cash on Delivery is the payment method the shop wants to start with. The middleware change came from a dev error: `fetch failed` at `getRegionMap`, because nothing was listening on `localhost:9000`. That turned every page into an error.

## How

**Brand text in one place**: every page, the navbar, the footer and the metadata read from `BRAND`. Contact details left empty are simply not shown.

**Page titles**: the root layout sets a title template, so child pages only give their own name. The home page uses `title.absolute` to skip the template.

**Delivery time**: the picker loads open slots from `GET /store/delivery-slots` and saves the choice with `POST /store/carts/:id/delivery-slot`. The choice lives in the cart metadata (`delivery_date`, `delivery_slot`), which moves to the order. If saving fails, the picker goes back to the earlier choice and shows the error.

**Cash on Delivery**: no new payment code. The built in `pp_system_default` provider is only given a new title, icon and text.

**Middleware fallback**: failures are not cached, so the next request tries the backend again. Only the fetch is guarded, the rest of the file is the same.

## Decisions made

**Placeholders for brand assets.** The logo, favicon and share image are simple placeholder files, and `heroImage` is empty so the hero uses a green background. The owner has to supply the real logo and photos. WhatsApp, email and address are empty because they are not public. The phone number is the one on vegein.in. The trust point wording is a draft for the owner to approve.

**Home page products removed.** The featured products block was taken off the home page so the page is hero plus trust strip. The product lists are reachable from the store page and the collections. The `featured-products` component is left in the repo, only unused on the home page.

**Medusa colour override.** Instead of changing every component, the interactive colours are set once in `globals.css` for light mode only. Dark mode is not changed.

**Middleware swallows backend errors.** I picked a fallback to the default region over letting the request fail. The cost is that a missing backend is only visible in the server log until a page needs backend data.

## Bugs found while testing (already there before, not fixed here)

**1. Old default country in `site.ts`.** `DEFAULT_COUNTRY_CODE` still falls back to `dk` when `NEXT_PUBLIC_DEFAULT_REGION` is not set. It is fine while `.env.local` sets `in`, but a fresh setup without the env value would use `dk`.

## Notes

- `npx tsc --noEmit` passes in the frontend. Lint was not run in this session.
- The delivery picker needs the backend changes in `PR-backend.md` to be running. Without them it shows "Could not load delivery times".
- The middleware change was not exercised against a stopped backend in this session.
- No automated tests were added, the repo has none for these parts.

## Next steps

- Get the real logo, hero photo, WhatsApp number, email and address from the owner, then fill `brand.ts` and replace the placeholder files.
- Check the pages at phone, tablet and desktop width.
- Decide if the home page should show product blocks again below the trust strip.

## Test plan

**Automated tests:**
- [x] `npx tsc --noEmit`: passes with no errors.
- [ ] `npx eslint` on the changed files: not run in this session.

**Manual checks (local):**
- [ ] Browser tab titles on home, product, category, collection, account and checkout: not run.
- [ ] Home page shows the logo, hero, "Shop now" button and trust strip, and the footer has no Medusa or GitHub links: not run.
- [ ] Checkout delivery step: pick a date and slot, "Continue to payment" enables, the review step and order page show the same time: not run, needs the backend running.
- [ ] Checkout payment step shows "Cash on Delivery" and the order page shows the cash message: not run, needs the backend and the COD script from `PR-backend.md`.
- [ ] Stop the backend and open the storefront: pages should load without the `fetch failed` crash. Not run.
